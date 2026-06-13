# Odoo 19 Development Guide — Proyecto `pruebas_opencode`

## Module Structure
```
module_name/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── *.py
├── views/
│   ├── *.xml
├── security/
│   ├── ir.model.access.csv
├── data/              (demo data or noupdate=1 data)
├── controllers/       (only if needed)
├── static/src/js/     (only if needed)
```

## `__manifest__.py` — Key Fields
```python
{
    'name': 'Module Name',
    'version': '19.0.1.0.0',
    'category': 'Sales',
    'depends': ['sale', 'project'],
    'data': [
        'security/ir.model.access.csv',
        'views/module_views.xml',
    ],
    'installable': True,
    'auto_install': False,
    'license': 'LGPL-3',
}
```

## Models
- **New model**: `_name = 'my.model'`, `_description`, `_order`, `_sql_constraints`, optional `name_get()` override
- **Inherit model**: `_inherit = 'existing.model'` (no `_name`)
- **Transient (wizard)**: `models.TransientModel`
- **Computed fields**: `@api.depends(...)` with loop over `self`
- **Onchange**: `@api.onchange('field')` to clear dependent fields (e.g., `self.field = False`)

## Views — Odoo 19 Specifics
- **Use `<list>` instead of `<tree>`** for list/tree views
- **`view_mode`**: use `"list,kanban"` NOT `"tree,kanban"`
- **Button in header**: `<xpath expr="//header" position="inside">`
- **Stat button**: `<xpath expr="//div[@name='button_box']" position="inside">`
- **New page in notebook**: `<xpath expr="//notebook" position="inside">`
- **Form button calling method**: `<button name="method_name" string="Label" type="object" class="btn-secondary"/>`
- **Open new record form (modal)**: return `act_window` with `target='new'` + context defaults

## Common Methods Pattern

### Stat button with DB count (always correct, survives cache)
```python
task_count = fields.Integer(compute='_compute_task_count')

@api.depends('task_ids')
def _compute_task_count(self):
    records_data = self.env['project.task']._read_group(
        [('sale_order_id', 'in', self.ids)],
        ['sale_order_id'],
        ['sale_order_id'],
    )
    data = {r['sale_order_id'][0]: r['sale_order_id_count'] for r in records_data}
    for record in self:
        record.task_count = data.get(record.id, 0)
```

### Create task — create first, then open for editing
```python
def create_task(self):
    self.ensure_one()
    new_task = self.env['project.task'].create({
        'sale_order_id': self.id,
        'name': 'New Task',
    })
    return {
        'type': 'ir.actions.act_window',
        'view_mode': 'form',
        'res_model': 'project.task',
        'res_id': new_task.id,
        'target': 'new',
        'context': {'form_view_initial_mode': 'edit'},
    }
```

### Inherited model — invalidate parent cache on child create
```python
def create(self, vals_list):
    tasks = super().create(vals_list)
    parents = tasks.mapped('parent_id')   # change to the actual relation field
    if parents:
        parents.invalidate_cache(['child_count'])  # field name on parent
    return tasks
```

### Open related records list (stat button action)
```python
def action_view_tasks(self):
    return {
        "type": "ir.actions.act_window",
        "domain": [('sale_order_id', '=', self.id)],
        "view_mode": "list,kanban",
        "res_model": "project.task",
        "target": "current",
        "context": {'default_sale_order_id': self.id},
    }
```

### Open a wizard (TransientModel)
```python
def action_open_wizard(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'res_model': 'my.wizard',
        'view_mode': 'form',
        'target': 'new',
        'context': {'default_parent_id': self.id},
    }
```

## Security
```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_model_user,model.user,model_my_model,base.group_user,1,1,1,1
```

## Key Odoo 19 Differences from Odoo 15/16
| Feature | Old (≤16) | New (19) |
|---------|-----------|----------|
| List/tree view | `<tree>` | `<list>` |
| View mode | `tree,form` | `list,form` |
| View mode | `tree,kanban` | `list,kanban` |

## Reference Modules (in this repo)
- `purchase_board_wizard/` — module with wizard, inherited models, Odoo 19 `<list>` syntax
- `sale_task_creator/` — module that creates tasks from sale orders with stat button and DB-count pattern
- `../carlos_dev_19/cd_custom_task/` — module that creates tasks from sale orders and CRM leads (Odoo 19)
