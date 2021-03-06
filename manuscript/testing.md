# Testing Ember.js applications

In this chapter we'll cover the basics of unit and acceptance testing in
Ember.js applications and recommend a couple of resources that can
help us expand our knowledge in this area.

## Unit Testing

When we run the generators, they create unit test files by default.
We can view all the generated unit tests if we go to `tests/unit`:

{title="Unit tests", lang="bash"}
~~~~~~~~
$ ls tests/unit/
adapters	controllers	models		utils
components	helpers		routes
~~~~~~~~

Tests are automatically grouped by type. If we open the unit test for our
friend model, we'll see the following:

{title="tests/unit/models/friend-test.js", lang="JavaScript"}
~~~~~~~~
import { test, moduleForModel } from 'ember-qunit';

moduleForModel('friend', 'Friend', { needs: ['model:article'] });

test('it exists', function(assert) {
  var model = this.subject();
  assert.ok(model);
});
~~~~~~~~

At the beginning of the test we import a set of helpers from
[ember-qunit](https://github.com/rwjblue/ember-qunit), which is a
library that wraps a bunch of functions to facilitate testing with
**QUnit**.

`moduleForModel` received the name of the model we are testing, a
description, and some options. In our scenario, we specify
that the tests need a model called **article** because of the existing
relationship between them.

Next, the test includes a basic assertion that the model exists.
`this.subject()` would be an instance of a `friend`.

We have two ways of running tests. The first one is via the browser while
we run the development server. We can navigate to
[http://localhost:4200/tests](http://localhost:4200/tests) and our
tests will be run. The second method is using a tests runner. At the
moment **ember-cli** has built-in support for **Testem** with
[PhantomJS](http://phantomjs.org/), which we can use to run our tests on
a CI server. To run tests in this mode, we only need to do `ember
test`.

I> We can also run tests with the command `npm test` which is aliased
I> to `ember test` in `package.json`.

Let's write two more tests for our friend model. We want to check that
the computed property `fullName` behaves as expected and that the
relationship articles is properly set.

{title="tests/unit/models/friend-test.js", lang="JavaScript"}
~~~~~~~~
import { test, moduleForModel } from 'ember-qunit';
import Ember from 'ember';

moduleForModel('friend', 'Friend', {
  needs: ['model:article']
});

test('it exists', function(assert) {
  var model = this.subject();
  assert.ok(model);
});

test('fullName joins first and last name', function(assert) {
  var model = this.subject({firstName: 'Syd', lastName: 'Barrett'});

  assert.equal(model.get('fullName'), 'Syd Barrett');

  Ember.run(function() {
    model.set('firstName', 'Geddy');
  });

  assert.equal(model.get('fullName'), 'Geddy Barrett', 'Updates fullName');
});

test('articles relationship', function(assert) {
  var klass  = this.subject({}).constructor;

  var relationship = Ember.get(klass, 'relationshipsByName').get('articles');

  assert.equal(relationship.key, 'articles');
  assert.equal(relationship.kind, 'hasMany');
});
~~~~~~~~

We can run our tests by going directly to the following URL:
[http://localhost:4200/tests?module=Friend](http://localhost:4200/tests?module=Friend).

The first test verifies that `fullName` is calculated
correctly. We have to wrap `model.set('firstName', 'Geddy');` in
`Ember.run` because it has an asynchronous behavior. If we modify the
implementation for `fullName` such that it doesn't return first and last
names, the tests will fail.

The second test checks that we have set up the proper
relationship to `articles`. Something similar could go in the articles
model tests. If we call `constructor` on an instance to a model, that
will give us access to the class of which it is an instance.


Let's add other unit test for `app/utils/date-helpers`:

{title="tests/unit/utils/date-helpers-test.js", lang="JavaScript"}
~~~~~~~~
import { module, test } from 'ember-qunit';
import dateHelpers from '../../../utils/date-helpers';

module('Utils: formatDate');

test('formats a date object', function(assert) {
  var date = new Date("11-3-2015");
  var result = dateHelpers.formatDate(date, 'ddd MMM DD YYYY');

  assert.equal(result, 'Mon Nov 03 2014', 'returns a readable string');
});
~~~~~~~~

We import the function we want to test and then check that
it returns the date as a readable string. We can run the test by going to
[http://localhost:4200/tests?module=Utils%3A%20formatDate](http://localhost:4200/tests?module=Utils%3A%20formatDate).

## Acceptance Tests

With acceptance tests we can verify workflows in our application. For
example, making sure that we can add a new friend, that if we visit the
friend index a list is rendered, etc. An acceptance test basically emulates a
real user's experience of our application.


Ember has a set of helpers to simplify writing these kinds of
tests. There are
[synchronous](http://emberjs.com/guides/testing/test-helpers/#toc_wait-helpers)
and
[asynchronous](http://emberjs.com/guides/testing/test-helpers/#toc_asynchronous-helpers)
helpers. We use the former for tests that don't have any kind of
side-effect, such as checking if an element is present on a page, and the
latter for tests that fire some kind of side-effect. For example,
clicking a link or saving a model.

Let's write an acceptance test to verify that we can add new friends
to our application. We can generate an acceptance test with the
generator **acceptance-test**.

~~~~~~~~
$ ember g acceptance-test friends/new
installing
  create tests/acceptance/friends/new-test.js
~~~~~~~~

If we visit the generated test, we'll see the following:

{title="tests/acceptance/friends/new-test.js", lang="JavaScript"}
~~~~~~~~
import Ember from 'ember';
import { module, test } from 'ember-qunit';
import startApp from '../helpers/start-app';

var application;

module('Acceptance: FriendsNew', {
  beforeEach: function() {
    application = startApp();
  },
  afterEach: function() {
    Ember.run(application, 'destroy');
  }
});

test('visiting /friends/new', function(assert) {
  visit('/friends/new');

  andThen(function() {
    assert.equal(currentPath(), 'friends/new');
  });
});
~~~~~~~~

We need to replace `import startApp from '../helpers/start-app';` with
`import startApp from '../../helpers/start-app';` and then make the
assertion of `currentPath` look for `friends.new` instead of
`friends/new`.

Now we can run our tests by visiting
[http://localhost:4200/tests](http://localhost:4200/tests) or, if we
want to run only the acceptance tests for Friends New,
[http://localhost:4200/tests?module=Acceptance%3A%20FriendsNew](http://localhost:4200/tests?module=Acceptance%3A%20FriendsNew).

Let's add two more tests but this time starting from the index URL. We
want to validate that we can navigate to new and then check that it
redirects to the correct place after creating a new user.

{title="Tests new friend: tests/acceptance/friends/new-test.js", lang="JavaScript"}
~~~~~~~~
test('Creating a new friend', function(assert) {
  visit('/');
  click('a[href="/friends/new"]');
  andThen(function() {
    assert.equal(currentPath(), 'friends.new');
  });
  fillIn('input[placeholder="First Name"]', 'Johnny');
  fillIn('input[placeholder="Last Name"]', 'Cash');
  fillIn('input[placeholder="email"]', 'j@cash.com');
  fillIn('input[placeholder="twitter"]', 'jcash');
  click('input[value="Save"]');

  //
  // Clicking save will fire an async event.
  // We can use andThen, which will be called once the promises above
  // have been resolved.
  //

  andThen(function() {
    assert.equal(
      currentRouteName(),
      'friends.show.index',
      'Redirects to friends.show after create'
    );
  });

});
~~~~~~~~

The second test we want to add checks that the application stays on the
new page if we click save, without adding any fields, and that an error
message is displayed:

{title="Tests new friend: tests/acceptance/friends/new-test.js", lang="JavaScript"}
~~~~~~~~
test('Clicking save without filling fields', function(assert) {
  visit('/friends/new');
  click('input[value="Save"]');
  andThen(function() {
    assert.equal(
      currentRouteName(),
      'friends.new',
      'Stays on new page'
    );
    assert.equal(
      find("h2:contains(You have to fill all the fields)").length,
      1,
      "Displays error message"
    );
  });

});
~~~~~~~~

### Mocking the API response

On the previous tests we hit the API, but this is not a common
scenario. Normally we'd like to mock the interactions with the API. To
do so we have different alternatives. One is to use
[Pretender](https://github.com/trek/pretender), a library that allows
us to mock requests with a simple DSL.

Another alternative is to use the
[built-in mock generator](http://www.ember-cli.com/#mocks-and-fixtures)
in **ember-cli**. This basically takes advantage of the Express server
used for development and extends it to capture requests to our API
end-points. With this tool, we can control what we would like to return for
each request.

Let's create a mock for `api/articles`:

~~~~~~~~
$ ember g http-mock articles
installing
  create server/.jshintrc
  create server/index.js
  create server/mocks/articles.js
  install package connect-restreamer
~~~~~~~~

If we open the generated file `server/mocks/articles.js`, we'll see the
following:

{title="server/mocks/articles.js", lang="JavaScript"}
~~~~~~~~
module.exports = function(app) {
  var express = require('express');
  var articlesRouter = express.Router();
  articlesRouter.get('/', function(req, res) {
    res.send({"articles":[]});
  });
  app.use('/api/articles', articlesRouter);
};

~~~~~~~~~

This intercepts the call to any request starting with `/api/articles`.
If it is a GET to `/`, it will return `{"articles":[]}`.

Suppose we want to mock the request for a particular article. We can
add the following:

{title="server/mocks/articles.js", lang="JavaScript"}
~~~~~~~~
module.exports = function(app) {
  var express = require('express');
  var articlesRouter = express.Router();
  articlesRouter.get('/', function(req, res) {
    res.send({"articles":[]});
  });
  articlesRouter.get('/articles/74', function(req, res) {
    res.send({
      "article":{
        "id":74,
        "created_at":"2014-11-03T21:30:47.869Z",
        "description":"foo",
        "state":"borrowed",
        "notes":"bar",
        "friend_id":153
      }
    });
  });
  app.use('/api/articles', articlesRouter);
};
~~~~~~~~

This will intercept any GET request to `/articles/74` and return the
mocked article.

## Further Reading

During EmberConf 2014, [Eric Berry](https://twitter.com/coderberry)
gave a great talk called
[The Unofficial, Official Ember Testing Guide](http://www.confreaks.com/videos/3310-emberconf2014-the-unofficial-official-ember-testing-guide)
where he walked us through testing in Ember.js. Eric also contributed an
excellent guide for testing that is now the official guide on the Ember.js
website. We recommend the official guide, which provides a complete
overview from unit to acceptance testing:
[http://emberjs.com/guides/testing/](http://emberjs.com/guides/testing).

To know more about using mocks and fixtures, we recommend the following
presentation:
[Real World Fixtures](https://speakerdeck.com/cball/real-world-fixtures)
by [Chris Ball](https://twitter.com/cball_).
