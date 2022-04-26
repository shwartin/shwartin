# Pytest quick start

## Fixture calculation

```
import random

@pytest.fixture
def rnd_gen():
    return random.Random(12345)

@pytest.fixture
def rnd(rnd_gen):
    return rnd_gen.random()

@pytest.fixture
def fixture_a(rnd):
    return rnd

@pytest.fixture
def fixture_b(rnd):
    return rnd

def test(fixture_a, fixture_b):
    assert fixture_a == fixture_b
```

Result will be the same. rnd call once.

## Fixture factories

```
@pytest.fixture
def make_rnd(rnd_gen):
    def maker():
        retrun rnd_gen.random()
    return maker

@pytest.fixture
def fixture_a(rnd):
    return rnd()

@pytest.fixture
def fixture_b(rnd):
    return rnd()

def test(fixture_a, fixture_b):
    assert fixture_a == fixture_b
```

Result will be different.

## Resource cleanup

```
@pytest.fixtures
def opened_file():
    f = open("filename"):
    try:
        yield f
    finally:
        f.close()

def test_a(opened_file):
    assert opened_file.read() == "file content"
```

Filename is fix.

## Cleanup for factories

```
@pytest.fixtures
def open_file():
    f = None
    def opener(filename):
        nonlocal f
        assert f is None
        f = open(filename)
        return f
    yield opener
    if f is not None:
        f.close()

def test_a(open_file):
    assert open_file("file_a.txt").read() == "Content A"
    assert open_file("file_b.txt").read() == "Content B"
```

Filename will be any.

## Fixtures for any context

```
@pytest.fixtures
def redis():
    with Redis() as redis:
        yield redis

@pytest.fixtures
def db():
    with sqlite3.connect(":memory:") as db:
        yield db

def test_a(db, redis):
    db.execute(...)
    redis.set(...)
```

## Transactions

```
@pytest.fixtures
def transaction(db):
    cursor = db.cursor()
    try:
        yield cursor
        db.commit()
    except:
        db.rollback()

def test_b(transaction):
    transaction.execute("INSERT INTO tbl(name) VALUES ('John')")
    raise RuntimeError("Error text")
    transaction.execute("INSERT INTO tbl(name) VALUES ('Mary')")
```

## Project structure

```
- root
    - project
        - __init__.py
        - ...
    - tests
        - conftest.py
        - redis_fixtures.py
        - db_fixtures.py
        - test_a.py
    - setup.py
```

No __init__.py in tests folder.

File `conftest.py`:

```
import pytest

pytest_plugins = ['redis_fixtures',  'db_fixtures']

@pytest.fixture
def fixture_a():
    return 'value'
```

No  fixture import in `conftest.py`

## Fixture scope

- function (default)
- class
- module
- session

## Unique id and docker client

```
import docker as libdocker

@pytest.fixture(scope='session')
def session_id():
    return str(uuid.uuid4())

@pytest.fixture(scope='session')
def unused_port():
    def f():
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.bind("127.0.0.1", 0)
            return s.getsockname()[1]
    return f

@pytest.fixture(scope='session')
def docker():
    return libdocker.Client(version='auto')
```

## Running container

```
@pytest.fixture(scope='session')
def redis_server(unused_port, session_id, docker):
    docker.pull('redis')
    port = unused_port()
    container = docker.create_container(
        image='redis',
        name=f'test-redis-{sission_id}',
        ports=[6379], detach=True,
        host_config=docker.create_host_config(
            port_bindings={6379: port}
        )
    )
    docker.start(container=container['Id']
    ping_redis(port)
    container['redis_port'] = port
    yield container
    docker.kill(container=container['Id'])
    docker.remove_container(container['Id'])
```

Run just one container for one test.

```
def ping_redis(port):
    timeout = 0.001
    for i in range(100):
        try:
            client = redis.StrictRedis(host="127.0.0.1", port=port, db=0)
        except redis.ConnectionError:
            time.sleep(timeout)
            timeout *= 2
        else:
            client.close()
    else:
        raise RuntimeError("Cannot connect to redis)
```

Need to whait while server is running up.

## Redis client

```
@pytest.fixture
def redis_client(redis_server):
    client = redis.StrictRedis(host="127.0.0.1", port=redis_server['redis_port'], db=0)
    yield client
    client.flushdb()
    client.close()

def test_redis(redis):
    redis_client(b'key', b'value')
    assert redis_client.get(b'key') == b'value'
```

Client need to watch how our code change redis.

## Test skipping

```
@pytest.mark.skipif(sys.version_info < (3, 4, 1),
    reason="Python<3.4.1 doesnt support __del__ callc from GC"
    )

def test__del__():
    ...
```

Run test on python >= 3.4.1

## Plugins

## Test exclusion

```
def pytest_ignore_collect(path, config):
    if 'py35' in str(path):
        if sys.version < (3, 5, 0):
            return True
```

```
- tests
    - conftest.py
    - ...
    - py35
        - test_1.py
        - test_2.py
```

Tests run only in python >= 3.5

## Add new comand line argument

```
def pytest_addoption(parser):
    parser.addoption('--gc-collect', action='store_true', default=False, help="Perform GC collection after every test")

@pytest.mark.trylast
def pytest_runtest_teardown(item, nextitem):
    if item.config.getoption('--gc-collect'):
        gc.collect()
    return nextitem
```

Add option '--gc-collect' to command line interface.

## Fixture parametrisation

```
@pytest.fixtures(scope="session", params=['2.8', '3.0'])
def redis_server(unused_port, session_id, docker, request):
    redis_version = request.param
    image = f'redis:{redis_version}'
    docker.pull(image)
    container = docker.create_container(
        image=image,
        name=f'test-redis-{redis_version}-{session_id},
        ...
    )
    ...
```

## Fixture generation

```
def pytest_addoption(parser):
    parser.addoption("--redis_version, action="append", default=[],
        help=(
            "Redist server versions."
            "May be used several times. "
            "Available values: 2.8, 3.0, 3.2 all"
        ))

def pytest_generate_tests(metafunc):
    if 'redis_version' in metafunc.fixturenames:
        tags = set(metafunc.config.option.redis_version)
        if not tags:
            tags = ['3.2']
        elif 'all' in tags:
            tags = ['2.8', '3.0', '3.2']
        else:
            tags = list(tags)
        metafunc.parametrize("redis_version", tags, scope='session')

@pytest.fixtures(scope="session"])
def redis_server(unused_port, session_id, docker, redis_version):
    image = f'redis:{redis_version}'
    docker.pull(image)
    container = docker.create_container(
        image=image,
        name=f'test-redis-{redis_version}-{session_id},
        ...
    )
    ...

```

## Skip long-runnig test

```
def pytest_addoption(parser):
    parser.addoption('--run-clow', action='store_true',
        default=False, help="Run slow tests")

def pytest_runtest_setup(item):
    if (
        'slowtest' in item.keywords and
        (not item.config.getoption('--run-slow'))
    ):
        pytest.skip("Need --run-slow to run")

@pytest.mark.slowtest
def test_xxx():
    ...
```

## Referencies

- https://www.youtube.com/watch?v=7KgihdKTWY4