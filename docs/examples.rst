Examples
========


Simple usage
------------

.. code-block:: python

    from django.db.models import Avg, Q
    from django_mock_queries.query import MockSet, MockModel

    qs = MockSet(
        MockModel(mock_name='john', email='john@gmail.com'),
        MockModel(mock_name='jeff', email='jeff@hotmail.com'),
        MockModel(mock_name='bill', email='bill@gmail.com'),
    )

    print [x for x in qs.all().filter(email__icontains='gmail.com').select_related('address')]
    # Outputs: [john, bill]

    qs = MockSet(
        MockModel(mock_name='model s', msrp=70000),
        MockModel(mock_name='model x', msrp=80000),
        MockModel(mock_name='model 3', msrp=35000),
    )

    print qs.all().aggregate(Avg('msrp'))
    # Outputs: {'msrp__avg': 61666}

    qs = MockSet(
        MockModel(mock_name='model x', make='tesla', country='usa'),
        MockModel(mock_name='s-class', make='mercedes', country='germany'),
        MockModel(mock_name='s90', make='volvo', country='sweden'),
    )

    print [x for x in qs.all().filter(Q(make__iexact='tesla') | Q(country__iexact='germany'))]
    # Outputs: [model x, s-class]

    qs = MockSet(cls=MockModel)
    print qs.create(mock_name='my_object', foo='1', bar='a')
    # Outputs: my_object

    print [x for x in qs]
    # Outputs: [my_object]

Test function that uses Django QuerySet:
----------------------------------------

.. code-block:: python

    """
    Function that queries active users
    """
    def active_users(self):
        return User.objects.filter(is_active=True).all()

    """
    Test function applies expected filters by patching Django's user model Manager or Queryset with a MockSet
    """
    from django_mock_queries.query import MockSet, MockModel


    class TestApi(TestCase):
        users = MockSet()
        user_objects = patch('django.contrib.auth.models.User.objects', users)

        @user_objects
        def test_api_active_users_filters_by_is_active_true(self):
            self.users.add(
                MockModel(mock_name='active user', is_active=True),
                MockModel(mock_name='inactive user', is_active=False)
            )

            for x in self.api.active_users():
                assert x.is_active

Test django-rest-framework model serializer
-------------------------------------------

.. code-block:: python

    """
    Car model serializer that includes a nested serializer and a method field
    """
    class CarSerializer(serializers.ModelSerializer):
        make = ManufacturerSerializer()
        speed = serializers.SerializerMethodField()

        def get_speed(self, obj):
            return obj.format_speed()

        class Meta:
            model = Car
            fields = ('id', 'make', 'model', 'speed',)

    """
    Test serializer returns fields with expected values and mock the result of nested serializer for field make
    """
    def test_car_serializer_fields(self):
        car = Car(id=1, make=Manufacturer(id=1, name='vw'), model='golf', speed=300)

        values = {
            'id': car.id,
            'model': car.model,
            'speed': car.formatted_speed(),
        }

        assert_serializer(CarSerializer) \
            .instance(car) \
            .returns('id', 'make', 'model', 'speed') \
            .values(**values) \
            .mocks('make') \
            .run()

Full Example
------------

There is a full Django application in the `examples/users` folder. It shows how
to configure `django_mock_queries` in your tests and run them with or without
setting up a Django database. Running the mock tests without a database can be
much faster when your Django application has a lot of database migrations.

To run your Django tests without a database, add a new settings file, and call
`monkey_patch_test_db()`. Use a wildcard import to get all the regular settings
as well.

.. code-block:: python

    # settings_mocked.py
    from django_mock_queries.mocks import monkey_patch_test_db

    from users.settings import *

    monkey_patch_test_db()

Then run your Django tests with the new settings file: ::

    ./manage.py test --settings=users.settings_mocked

Here's the pytest equivalent: ::

    pytest --ds=users.settings_mocked

That will run your tests without setting up a test database. All of your tests
that use Django mock queries should run fine, but what about the tests that
really need a database? ::

    ERROR: test_create (examples.users.analytics.tests.TestApi)
    ----------------------------------------------------------------------
    Traceback (most recent call last):
      File "/.../examples/users/analytics/tests.py", line 28, in test_create
        start_count = User.objects.count()
      [...]
    NotSupportedError: Mock database tried to execute SQL for User model.

If you want to run your tests without a database, you need to tell Django
to skip the tests that need a database. You can do that by putting a skip
decorator on the test classes or test methods that need a database.

.. code-block:: python

    @skipIfDBFeature('is_mocked')
    class TestApi(TestCase):
        def test_create(self):
            start_count = User.objects.count()

            User.objects.create(username='bob')
            final_count = User.objects.count()

            self.assertEqual(start_count + 1, final_count)
