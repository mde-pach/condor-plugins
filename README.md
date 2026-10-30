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
| `used` | functions, methods and classes matching `decorated_by`, `names`, `in_classes_inheriting` or `in_classes_based_on` (methods), or `inherits` (classes); every selector given must match |
| `fields_used` | attributes of classes matching any of `decorated_by`, `inherits` or `names`, and of their subclasses: `annotated` (`x: int`, the default), `assigned` (`X = 1`), `calls` (assigned a call, `name = models.CharField()`) or `all`, given as `fields`; `attributes` limits the rule to these names (a framework's options) |
| `parameters_used` | the parameters of functions matching `decorated_by`, `names`, `in_classes_inheriting` or `in_classes_based_on`; every selector given must match |
| `injected_by_name` | providers (fixtures) requested through consumers' parameter names, a decorator keyword, or calls such as `usefixtures` |
| `variables_used` | module and class variables with these `names` (`pytestmark`), in files that import the library |

`in_classes_inheriting` follows the class hierarchy across the project;
`in_classes_based_on` only looks at a class's own bases (a `Protocol` is a
class listing `Protocol` itself; its concrete subclasses are not protocols).
Bases are resolved through each file's imports, so an unrelated class with
the same name elsewhere does not match.

Rules without a class hierarchy to follow (decorator patterns such as
`*.get`, class `names` such as `Meta`) apply in files that import a
`when_imported` module, and to decorators whose object is imported
(`from myapp import app` then `@app.route`).

Selector lists must not be empty (leave the key out instead), and
`when_imported` takes top-level module names. JSON plugins may set
`"$schema"` for editor support.

Names are dotted names as imported (`pydantic.BaseModel`, not `BaseModel`),
with glob wildcards. condor resolves names through each file's imports.
Decorators on local objects keep their written name (`register.filter`,
`*.route`).

## Plugins

| File | Library |
|---|---|
| `python/abc.toml`, `dataclasses.toml`, `enum.toml`, `typing.toml`, `unittest.toml` | standard library |
| `python/pytest.toml` | pytest, pytest-asyncio |
| `python/pydantic.toml`, `attrs.toml`, `msgspec.toml`, `sqlmodel.toml` | data classes and models |
| `python/django.toml`, `celery.toml`, `flask.toml`, `fastapi.toml`, `click.toml` | frameworks |

## Tests

Each plugin has cases under `tests/python/<plugin>/<case>/`: a directory of
Python files that condor scans as one project. A line that must be reported
ends with `# dead: name` (several names separated by commas), on the line
condor reports (a decorated definition's first decorator); every other
definition must not be reported. A case shows what a rule declares used next
to its closest look-alikes that must still be reported: a helper method in a
model, the same decorator on a local object, a same-named class elsewhere,
the library not imported. condor runs every case in its test suite.

## Contributing

Add or change a file under `python/`, explain each rule with a comment, and
add cases for it under `tests/`. Then check it with condor on a project that
uses the library: the findings that disappear should all be false positives.
