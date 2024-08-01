# CHAPTER 2 Writing Test Functions

`pytest test_card.py`

`pytest -vv test_card_fail.py`

### Failing with pytest.fail() and Exceptions

        def test_with_fail():
            c1 = Card("sit there", "brian")
            c2 = Card("do something", "okken")
            if c1 != c2:
                pytest.fail("they don't match")

### Writing Assertion Helper Functions

        def assert_identical(c1: Card, c2: Card):
            __tracebackhide__ = True
            assert c1 == c2
            if c1.id != c2.id:
                pytest.fail(f"id's don't match. {c1.id} != {c2.id}")
        def test_identical():
            c1 = Card("foo", id=123)
            c2 = Card("foo", id=123)
            assert_identical(c1, c2)

### Testing for Expected Exceptions

        def test_no_path_fail():
            cards.CardsDB()

`pytest --tb=short test_experiment.py`
`--tb=short` shorter traceback format because we don’t need to see the full traceback to find out which exception is raised.

        def test_raises_with_info():
            match_regex = "missing .* positional argument"
            with pytest.raises(TypeError, match=match_regex):
                cards.CardsDB()

        def test_raises_with_info_alt():
            with pytest.raises(TypeError) as exc_info:
                cards.CardsDB()
                expected = "missing 1 required positional argument"
                assert expected in str(exc_info.value)

### Structuring Test Functions

        def test_to_dict():
            # GIVEN a Card object with known contents
            c1 = Card("something", "brian", "todo", 123)
            # WHEN we call to_dict() on the object
            c2 = c1.to_dict()
            # THEN the result will be a dictionary with known content
            c2_expected = {
                "summary": "something",
                "owner": "brian",
                "state": "todo",
                "id": 123,
                }
            assert c2 == c2_expected

### Grouping Tests with Classes

        class TestEquality:
            def test_equality(self):
                c1 = Card("something", "brian", "todo", 123)
                c2 = Card("something", "brian", "todo", 123)
                assert c1 == c2
            def test_equality_with_diff_ids(self):
                c1 = Card("something", "brian", "todo", 123)
                c2 = Card("something", "brian", "todo", 4567)
                assert c1 == c2
            def test_inequality(self):
                c1 = Card("something", "brian", "todo", 123)
                c2 = Card("completely different", "okken", "done", 123)
                assert c1 != c2

`pytest -v test_classes.py::TestEquality`

###Running a Subset of Tests

Running a single test method, test class, or module:
`$ pytest ch2/test_classes.py::TestEquality::test_equality`
`$ pytest ch2/test_classes.py::TestEquality`
`$ pytest ch2/test_classes.py`
Running a single test function or module:
`$ pytest ch2/test_card.py::test_defaults`
`$ pytest ch2/test_card.py`
Running the whole directory:
`$ pytest ch2`

`$ pytest -v -k TestEquality`
use `-k` and just specify the test class name

`pytest -v --tb=no -k equality`
run all the tests with “equality” in their name
`pytest -v --tb=no -k "equality and not equality_fail"`
keywords and, not, or, and parentheses are allowed to create complex expressions
`pytest -v --tb=no -k "(dict or ids) and not TestEquality"`
test run of all tests with “dict” or “ids” in the name, but not ones in the “TestEquality” class

## CHAPTER 3 pytest Fixtures

        @pytest.fixture()
        def cards_db():
            with TemporaryDirectory() as db_dir:
                db_path = Path(db_dir)
                db = cards.CardsDB(db_path)
                yield db
                db.close()
        def test_empty(cards_db):
            assert cards_db.count() == 0
        def test_two(cards_db):
            cards_db.add_card(cards.Card("first"))
            cards_db.add_card(cards.Card("second"))
            assert cards_db.count() == 2

### Tracing Fixture Execution with –setup-show

`$ pytest --setup-show test_count.py`

### Specifying Fixture Scope

`@pytest.fixture(scope="module")`

`scope='function'`
`scope='class'`
`scope='module'`
`scope='package'`
`scope='session'`

### Sharing Fixtures through conftest.py

### Finding Where Fixtures Are Defined

`$ pytest --fixtures -v`

`pytest --fixtures-per-test test_count.py::test_empty`
use --fixtures-per-test to see what fixtures are used by each test and where the fixtures are defined

### Using Multiple Fixture Levels

        @pytest.fixture(scope="session")
        def db():
            """CardsDB object connected to a temporary database"""
            with TemporaryDirectory() as db_dir:
                db_path = Path(db_dir)
                db_ = cards.CardsDB(db_path)
                yield db_
                db_.close()
        @pytest.fixture(scope="function")
        def cards_db(db):
            """CardsDB object that's empty"""
            db.delete_all()
            return db

### Using Multiple Fixtures per Test or Fixture

        @pytest.fixture(scope="session")
        def some_cards():
            """List of different Card objects"""
            return [
                cards.Card("write book", "Brian", "done"),
                cards.Card("edit book", "Katie", "done"),
                cards.Card("write 2nd edition", "Brian", "todo"),
                cards.Card("edit 2nd edition", "Katie", "todo"),
            ]

        def test_add_some(cards_db, some_cards):
            expected_count = len(some_cards)
            for c in some_cards:
            cards_db.add_card(c)
            assert cards_db.count() == expected_count


        @pytest.fixture(scope="function")
        def non_empty_db(cards_db, some_cards):
            """CardsDB object that's been populated with 'some_cards'"""
            for c in some_cards:
                cards_db.add_card(c)
                return cards_db

        def test_non_empty(non_empty_db):
            assert non_empty_db.count() > 0

### Deciding Fixture Scope Dynamically

        @pytest.fixture(scope=db_scope)
        def db():
            """CardsDB object connected to a temporary database"""
            with TemporaryDirectory() as db_dir:
                db_path = Path(db_dir)
                db_ = cards.CardsDB(db_path)
                yield db_
                db_.close()

        def db_scope(fixture_name, config):
            if config.getoption("--func-db", None):
                return "function"
            return "session"

        def pytest_addoption(parser):
            parser.addoption(
                "--func-db",
                action="store_true",
                default=False,
                help="new db for each test",
            )

`pytest --setup-show test_count.py`
`pytest --func-db --setup-show test_count.py`

### Using autouse for Fixtures That Always Get Used

        import pytest
        import time
        @pytest.fixture(autouse=True, scope="session")
        def footer_session_scope():
            """Report the time at the end of a session."""
            yield
            now = time.time()
            print("--")
            print(
            "finished : {}".format(
            time.strftime("%d %b %X", time.localtime(now))
            )
            )
            print("-----------------")
        @pytest.fixture(autouse=True)
        def footer_function_scope():
            """Report test durations after each function."""
            start = time.time()
            yield
            stop = time.time()
            delta = stop - start
            print("\ntest duration : {:0.3} seconds".format(delta))
        def test_1():
            """Simulate long-ish running test."""
            time.sleep(1)
        def test_2():
            """Simulate slightly longer test."""
            time.sleep(1.23)

`pytest -v -s test_autouse.py`
-s flag in this example. It’s a shortcut flag for --capture=no that tells pytest to turn off output capture

### Renaming Fixtures

        import pytest
        @pytest.fixture(name="ultimate_answer")
        def ultimate_answer_fixture():
            return 42
        def test_everything(ultimate_answer):
            assert ultimate_answer == 42

## CHAPTER 4 Builtin Fixtures

### Using tmp_path and tmp_path_factory

def test_tmp_path(tmp_path):
    file = tmp_path / "file.txt"
    file.write_text("Hello")
    assert file.read_text() == "Hello"

def test_tmp_path_factory(tmp_path_factory):
    path = tmp_path_factory.mktemp("sub")
    file = path / "file.txt"
    file.write_text("Hello")
    assert file.read_text() == "Hello"

• tmp_path_factory is session scope.
• tmp_path is function scope.

