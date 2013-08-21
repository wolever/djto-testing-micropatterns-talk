Factories, not Fixtures
=======================

Using a Fixture
---------------

``some_specific_permission.yaml``::

    - model: auth.user
      pk: 64
      fields:
        username: some-specific-thing
        is_active: true
    - model: myapp.wikipage_permission
      pk: 16
      fields:
        user_id: 64
        permission: read

``test_permissions.py``::

    class TestWikiPagePermissions(TestCase):
        fixtures = ['some_specific_permission']

        def test_some_specific_thing(self):
            permission = Permission.objects.get(id=16)


Using a Factory
---------------

``test_permissions.py``::

    class TestWikiPagePermissions(TestCase):
        def test_some_specific_thing(self):
            perm = WikipagePermissionFactory(permission="read")


``factories.py``::

    WikipagePermissionFactory = Factory(WikiPagePermission)


Where do I Find this Magical Factory?
-------------------------------------

Check out ``model_mommy``: https://github.com/vandersonmota/model_mommy

::
    from model_mommy import mommy

    class Factory(object):
        def __init__(self, model, **attrs):
            self.model = model
            self.attrs = attrs

        def __call__(self, **custom_attrs):
            attrs = dict(self.attrs)
            attrs.update(custom_attrs)
            return mommy.make_one(self.model, **custom_attrs)


\o/

Keep ALL fields unique!
=======================

Unless a field **should** be the same between two instances, make sure it's
unique!

::
    gen = StringGenerator()
    Factory = FactoryBase.with_gen(gen)

    UserFactory = Factory(User,
        username=gen("user-{num}"),
        first_name=gen("Alex-{num}"),
        last_name=gen("Smith-{num}"),
        email=gen("user-{num}@example.com"),
    )

And it works like this::

    >>> u = UserFactory()
    >>> u.username
    'user-1'
    >>> u2 = UserFactory()
    'user-2'

What is this magic ``StringGenerator()``?

::
    from functools import partial

    class StringGenerator(object):
        def __init__(self):
            self.stack = []
            self.num = 0
            self.next = 1
            self.need_incr = True
            self.need_decr = False

        def incr(self):
            self.need_incr = True

        def decr(self):
            if self.need_decr:
                self.num = self.stack.pop()
                self.need_decr = False

        def _generator(self, s):
            if self.need_incr:
                self.need_incr = False
                self.stack.append(self.num)
                self.need_decr = True
                self.num = self.next
                self.next += 1
            return unicode(s).format(num=self.num)

        def __call__(self, s):
            return partial(self._generator, s)


    class FactoryBase(object):
        def __init__(self, gen, model, **attrs):
            self.model = model
            self.attrs = attrs

        @classmethod
        def with_gen(cls, gen):
            return partial(cls, gen)

        def prep_attrs(self, attrs):
            for key, val in attrs.items():
                if callable(val):
                    attrs[key] = val()
            return attrs

        def build_attrs(self, custom_attrs):
            attrs = dict(self.attrs)
            attrs.update(custom_attrs)
            self.gen.incr()
            attrs = self.prep_attrs(attrs)
            self.gen.decr()
            return attrs

        def __call__(self, **custom_attrs):
            return mommy.make_one(self.model, **self.build_attrs(custom_attrs))



Unicode ALL the things!
=======================

Make sure that **all** your string fields contain unicode:

::
    UserFactory = Factory(User,
        username=gen(u"üser-{num}"),
        first_name=gen(u"Ålex-{num}"),
        last_name=gen(u"Smi†h-{num}"),
        email=gen("user-{num}@example.com"),
    )
