# express-validator-helper

[![img](http://img.shields.io/npm/v/express-validator-helper.svg)](https://www.npmjs.com/package/express-validator-helper)
[![img](http://img.shields.io/npm/l/express-validator-helper.svg)](https://www.npmjs.com/package/express-validator-helper)
[![img](http://img.shields.io/github/stars/evilrobts/express-validator-helper.svg)](https://github.com/evilrobts/express-validator-helper)
[![img](http://img.shields.io/npm/dm/express-validator-helper.svg)](https://www.npmjs.com/package/express-validator-helper)
[![img](http://img.shields.io/travis/evilrobts/express-validator-helper.svg)](https://travis-ci.org/evilrobts/express-validator-helper)
[![img](http://img.shields.io/coveralls/evilrobts/express-validator-helper.svg)](https://coveralls.io/r/evilrobts/express-validator-helper)

Simple [express-validator](https://github.com/ctavan/express-validator) wrapper for easier work with validation error handling.

![img](./img.jpg)

## Installation

```bash
$ npm i express-validation-helper -S
```

```javascript
// add expressValidator before helper
app.use(expressValidator());
app.use(expressValidatorHelper());
```

## Before & After

Confirmation handler example.

### Before
```javascript
exports.create = function(req, res, next) {
  req.assert('email', 'invalid format').isEmail();
  req.assert('email', 'required').notEmpty();

  var errors = req.validationErrors(true);
  var email = req.body.email;

  return User.findOne({
    email: new RegExp('^' + email + '$', 'i')
  }, function(err, user) {
    if (err) { return next(err); }

    if (user && user.isEmailConfirmed === true && !errors.email) {
      errors = errors || {};
      errors.email = {
        param: 'email',
        msg: 'you account already confirmed',
        value: email
      };
    }

    if (!user && !errors.email) {
      errors = errors || {};
      errors.email = {
        param: 'email',
        msg: 'user with that email not found'
      };
    }

    if (!_.isEmpty(errors)) {
      return res.render('confirmations/new', {
        errors: errors,
        values: req.body
      });
    }

    // ... success
  });
};
```

### After
```javascript
exports.create = function(req, res, next) {
  req.assert('email', 'invalid format').isEmail();
  req.assert('email', 'required').notEmpty();

  var validator = req.validate();
  var email = req.body.email;

  return User.findOne({
    email: new RegExp('^' + email + '$', 'i')
  }, function(err, user) {
    if (err) { return next(err); }

    if (user && user.isEmailConfirmed === true) {
      validator.add('you account already confirmed');
    }

    if (!user && !errors.email) {
      validator.add('user with that email not found')
    }

    if (!validator.isValid()) {
      return res.render('confirmations/new', {
        errors: validator.errors,
        values: req.body
      });
    }

    // ... success
  });
};
```

## Reason
express-validator (Validator.js middleware for Express.js) is wonderful tool for validating user input. But it does not allow to define custom async validators. For example we want to check if email is already taken on signup page. Ok, we can add errors manually to  `req.validationErrors()` result. And here we are faced with several problems:

### `req.validationErrors()` result
express-validator provides two ways to get a result - as an array or object.

Array is not very useful if you want to display an error for each field separately. It is not for us.

Object contains only the last error:

```javascript
req.assert('email', __('required')).notEmpty();
req.assert('email', __('invalid format')).isEmail();

var errors = req.validationErrors(true);
```

Here the `errors` will contain message for format validation instead `notEmpty()` check. So you have to do the checking in reverse order. This is the first problem.

### Adding errors
`req.validationErrors(true)` can return `null` or `object` with errors and we should check it. Also error for a particular field may already be in the object and we should check it too:

```javascript
req.assert('email', __('invalid format')).isEmail();
req.assert('email', __('required')).notEmpty();

var errors = req.validationErrors(true);

User.findOne({ email: req.body.email }, function(err, existsUser) {
  if (err) { return next(err); }

  if (existsUser && !errors.email) {
    errors = errors || {};
    errors.email = {
      param: 'email',
      msg: 'already taken',
      value: req.body.email
    }
  }
});
```
If we do not check this, our error will appear even when unfilled or incorrect e-mail address.

It becomes routine, when we do more of checks like this.
