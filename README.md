# karbor-changes
Changes to fix Karbor

## karborclient

### container: karbor_operationengine
- source: https://github.com/openstack/python-karborclient/blob/stable/stein/karborclient/v1/checkpoints.py
- file location: `/var/lib/kolla/venv/lib/python2.7/site-packages/karborclient/v1/checkpoints.py`
```
    def create(self, provider_id, plan_id, checkpoint_extra_info=None):
        body = {'checkpoint': {'plan_id': plan_id,
 -                             'extra-info': checkpoint_extra_info}}
 +                             'extra_info': checkpoint_extra_info}}
```
&nbsp;
____________________________________________________________________________________________________

## Karbor 

### container: karbor_api

- source: https://github.com/openstack/karbor/blob/stable/stein/karbor/api/schemas/checkpoints.py
- file location: `/var/lib/kolla/venv/lib/python2.7/site-packages/karbor/api/schemas/checkpoints.py`
```
    create = {
        'type': 'object',
        'properties': {
            'type': 'object',
            'checkpoint': {
                'type': 'object',
                'properties': {
                    'plan_id': parameter_types.uuid,
 -                  'extra-info': parameter_types.metadata,
 +                  'extra_info': parameter_types.metadata,
```
&nbsp;

### container: karbor_operationengine

- source: https://github.com/openstack/karbor/blob/stable/stein/karbor/services/operationengine/user_trust_manager.py
- file location: `/var/lib/kolla/venv/lib/python2.7/site-packages/karbor/services/operationengine/user_trust_manager.py`
```
    def add_operation(self, context, operation_id):
        LOG.debug("adding operation with context %s", context)
        auth_info = self._get_user_trust_info(
            context.user_id, context.project_id)
 -      if auth_info:
 -          auth_info['operation_ids'].add(operation_id)
 -          return auth_info['trust_id']

        trust_id = self._skp.create_trust_to_karbor(context)
        try:
            lsession = self._skp.create_trust_session(trust_id)
        except Exception:
            self._skp.delete_trust_to_karbor(trust_id)
            raise
 -      self._add_user_trust_info(context.user_id, context.project_id,
 -                                operation_id, trust_id, lsession)
 +      if auth_info:
 +         auth_info['operation_ids'].add(operation_id)
 +         auth_info['session'] = lsession
 +         auth_info['trust_id'] = trust_id
 +      else:
 +         self._add_user_trust_info(context.user_id, context.project_id,
                                     operation_id, trust_id, lsession)

        return trust_id
```
&nbsp;

- source: https://github.com/openstack/karbor/blob/stable/stein/karbor/services/operationengine/operations/retention_operation.py
- file location: `/var/lib/kolla/venv/lib/python2.7/site-packages/karbor/services/operationengine/operations/retention_operation.py`
```
    def _run(self, operation_definition, param, log_ref):
        project_id = param.get("project_id")
        client = self._create_karbor_client(
            param.get("user_id"), project_id)
        provider_id = operation_definition.get("provider_id")
        plan_id = operation_definition.get("plan_id")
        trigger_id = param.get("trigger_id", None)
        scheduled_operation_id = param.get("scheduled_operation_id", None)
        extra_info = {
            'created_by': constants.OPERATION_ENGINE,
            'trigger_id': trigger_id,
            'scheduled_operation_id': scheduled_operation_id
        }
 -      LOG.debug("Create checkpoint: provider_id=%(provider_id)s, "
 -                "plan_id=%(plan_id)s, trigger_id=%(trigger_id)s, "
 -                "scheduled_operation_id=%(scheduled_operation_id)s" %
 -                {"provider_id": provider_id,
 -                 "plan_id": plan_id,
 -                 "trigger_id": trigger_id,
 -                 "scheduled_operation_id": scheduled_operation_id})
 -      try:
 -           client.checkpoints.create(provider_id, plan_id, extra_info)
 -       except Exception:
 -           state = constants.OPERATION_EXE_STATE_FAILED
 -       else:
 -           state = constants.OPERATION_EXE_STATE_SUCCESS
 -       finally:
 -           self._update_log_when_operation_finished(log_ref, state)

 //...

        except Exception:
            state = constants.OPERATION_EXE_DURATION_STATE_FAILED
            reason = (_("Can't execute retention policy provider_id: "
                        "%(provider_id)s plan_id:%(plan_id)s"
                        " retention_duration:%(retention_duration)s") %
                      {"provider_id": provider_id, "plan_id": plan_id,
                       "retention_duration": retention_duration})
            raise exception.InvalidOperationDefinition(reason=reason)
        finally:
            self._update_log_when_operation_finished(log_ref, state)

 +      LOG.debug("Create checkpoint: provider_id=%(provider_id)s, "
 +                "plan_id=%(plan_id)s, trigger_id=%(trigger_id)s, "
 +                "scheduled_operation_id=%(scheduled_operation_id)s" %
 +                {"provider_id": provider_id,
 +                 "plan_id": plan_id,
 +                 "trigger_id": trigger_id,
 +                 "scheduled_operation_id": scheduled_operation_id})
 +      try:
 +          client.checkpoints.create(provider_id, plan_id, extra_info)
 +      except Exception:
 +          state = constants.OPERATION_EXE_STATE_FAILED
 +      else:
 +          state = constants.OPERATION_EXE_STATE_SUCCESS
 +      finally:
 +          self._update_log_when_operation_finished(log_ref, state)

 //...

    def _delete_old_backup_by_max_backups(
            self, client, max_backups, project_id, provider_id, plan_id):

        if max_backups == -1:
            return

        backup_items = self._list_available_checkpoint(
            client, project_id, provider_id, plan_id)
        count = len(backup_items)
        LOG.debug("Nummber of backup items: %s", str(count))
        if count > max_backups:
 -          for item in backup_items[max_backups:]:
 +          for item in backup_items[max_backups-1:]:
                try:
                    client.checkpoints.delete(provider_id, item.id)
                except Exception as e:
                    reason = (_("Failed to delete checkpoint: %(cp_id)s by "
                                "max_backups with the reason: %(reason)s") %
                              {"cp_id": item.id, "reason": e})
                    raise exception.InvalidOperationDefinition(reason=reason)
```
&nbsp;

### container: karbor_protection