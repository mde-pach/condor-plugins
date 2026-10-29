# condor-plugins

Rules that tell [condor](https://github.com/mde-pach/condor), the Python
dead-code finder, how libraries use code: which methods a framework calls,
which class attributes are fields, which functions it passes by name. Without
them, that code looks unused.

condor ships a pinned copy of this repository. Projects can add their own
rule files with `[tool.condor] plugins = ["path/to/rules.toml"]`.

## Format

A plugin is a TOML or JSON file following [schema.json](schema.json):

```toml
[plugin]
name = "pydantic"
when_imported = ["pydantic"]      # applies when the project imports pydantic

[[fields_used]]                   # annotated attributes of models are used
inherits = ["pydantic.BaseModel"]
fields = "annotated"              # annotated | assigned | all

[[used]]                          # these definitions are used
decorated_by = ["pydantic.field_validator"]
```

| Rule | Declares used |
|---|---|
| `used` | functions, methods and classes matching `decorated_by`, `names`, `in_classes_inheriting` (methods) or `inherits` (classes); every selector given must match |
| `fields_used` | attributes of classes matching `decorated_by`, `inherits` or `names`, and of their subclasses |
| `parameters_used` | the parameters of functions matching `decorated_by`, `names` or `in_classes_inheriting` |
| `injected_by_name` | providers (fixtures) requested through consumers' parameter names, a decorator keyword, or calls such as `usefixtures` |

Names are dotted names as imported (`pydantic.BaseModel`, not `BaseModel`),
with glob wildcards. condor resolves names through each file's imports.
Decorators on local objects keep their written name (`register.filter`,
`*.route`).

## Plugins

| File | Library |
|---|---|
| `python/abc.toml`, `dataclasses.toml`, `enum.toml`, `typing.toml` | standard library |
| `python/pytest.toml` | pytest, pytest-asyncio |
| `python/pydantic.toml`, `attrs.toml`, `msgspec.toml`, `sqlmodel.toml` | data classes and models |
| `python/django.toml`, `celery.toml`, `flask.toml`, `fastapi.toml`, `click.toml` | frameworks |

## Contributing

Add or change a file under `python/`, explain each rule with a comment, and
check it with condor on a project that uses the library: the findings that
disappear should all be false positives.
