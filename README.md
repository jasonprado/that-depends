"That Depends"
==
[![PyPI version](https://badge.fury.io/py/that-depends.svg)](https://pypi.python.org/pypi/that-depends)
[![Supported versions](https://img.shields.io/pypi/pyversions/that-depends.svg)](https://pypi.python.org/pypi/that-depends)
[![GitHub license](https://img.shields.io/github/license/modern-python/that-depends)](https://github.com/modern-python/that-depends/blob/main/LICENSE)
[![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/modern-python/that-depends/python-package.yml)](https://github.com/modern-python/that-depends/actions)
[![Doc](https://readthedocs.org/projects/that-depends/badge/?version=latest&style=flat)](https://that-depends.readthedocs.io)
[![GitHub stars](https://img.shields.io/github/stars/modern-python/that-depends)](https://github.com/modern-python/that-depends/stargazers)

This package is dependency injection framework for Python and allows to transfer projects from `dependency-injector` with minimal costs and gain additional capabilities.

It is production-ready and gives you the following:
- Simple async-first DI framework with IOC-container.
- Python 3.10-3.12 support.
- Full coverage by types annotations (mypy in strict mode).
- FastAPI and LiteStar compatibility.
- Overriding dependencies for tests.
- Injecting dependencies in functions and coroutines without wiring.
- Package with zero dependencies.

📚 [Documentation](https://that-depends.readthedocs.io)

# Projects with `That Depends`:
- [fastapi-sqlalchemy-template](https://github.com/modern-python/fastapi-sqlalchemy-template) - FastAPI template with sqlalchemy2 and PostgreSQL
- [litestar-sqlalchemy-template](https://github.com/modern-python/litestar-sqlalchemy-template) - LiteStar template with sqlalchemy2 and PostgreSQL

# Quickstart
## Install
```bash
pip install that-depends
```

## Describe resources and classes:
```python
import dataclasses
import logging
import typing


logger = logging.getLogger(__name__)


# singleton provider with finalization
def create_sync_resource() -> typing.Iterator[str]:
    logger.debug("Resource initiated")
    try:
        yield "sync resource"
    finally:
        logger.debug("Resource destructed")


# same, but async
async def create_async_resource() -> typing.AsyncIterator[str]:
    logger.debug("Async resource initiated")
    try:
        yield "async resource"
    finally:
        logger.debug("Async resource destructed")


@dataclasses.dataclass(kw_only=True, slots=True)
class DependentFactory:
    sync_resource: str
    async_resource: str
```

## Describe IoC-container
```python
from that_depends import BaseContainer, providers


class DIContainer(BaseContainer):
    sync_resource = providers.Resource(create_sync_resource)
    async_resource = providers.AsyncResource(create_async_resource)

    simple_factory = providers.Factory(SimpleFactory, dep1="text", dep2=123)
    dependent_factory = providers.Factory(
        sync_resource=sync_resource,
        async_resource=async_resource,
    )
```

## Resolve dependencies in your code
```python
# async resolving by default:
await DIContainer.simple_factory()

# sync resolving is also allowed if there is no uninitialized async resources in dependencies
DIContainer.simple_factory.sync_resolve()

# otherwise you can initialize async resources beforehand one by one or in one call:
await DIContainer.init_async_resources()
```

## Resolve dependencies not described in container
```python
@dataclasses.dataclass(kw_only=True, slots=True)
class FreeFactory:
    dependent_factory: DependentFactory
    sync_resource: str

# this way container will try to find providers by names and resolve them to build FreeFactory instance
free_factory_instance = await DIContainer.resolve(FreeFactory)
```

## Inject providers in function arguments
```python
import datetime

from that_depends import inject, Provide

from tests import container


@inject
async def some_coroutine(
    simple_factory: container.SimpleFactory = Provide[container.DIContainer.simple_factory],
    dependent_factory: container.DependentFactory = Provide[container.DIContainer.dependent_factory],
    default_zero: int = 0,
) -> None:
    assert simple_factory.dep1
    assert isinstance(dependent_factory.async_resource, datetime.datetime)
    assert default_zero == 0

@inject
def some_function(
    simple_factory: container.SimpleFactory = Provide[container.DIContainer.simple_factory],
    default_zero: int = 0,
) -> None:
    assert simple_factory.dep1
    assert default_zero == 0
```
