## user.controller.js

> listAll: összes regisztrált felhasználó listázása
> register: felhasználó regisztrálása név, e-mail és jogosultsággal.
> update: a felhasználó adatainak frissítése. Új jelszó is lehetséges.
> getOne: adott id-jú felhasználó keresése.

```javascript
const User = require('../models/user');
/**
 * @module User
 */
module.exports = {
  profile: (req, res) => {
    res.json({
      user: req.user,
    });
  },

  listAll: (req, res) => {
    User.find({}, (err, user) => {
      if (err) {
        res.send(err);
      }
      res.json(user);
    });
  },


  register: (req, res) => {
    User.register(new User({
      username: req.body.username,
      email: req.body.email,
      rights: req.body.rights,
    }), req.body.password)
      .then(user => res.json(user))
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },

  login: (req, res) => res.json({
    login: true,
    user: req.body,
  }),

  logout: (req, res) => {
    req.logout();
    res.status(200).json({
      success: 'Sikeres kilépés',
    });
  },

  remove: (req, res) => {
    User.findByIdAndRemove(req.params.id)
      .then(() => {
        res.status(200).json({
          success: 'Sikeres törlés',
        });
      })
      .catch((err) => {
        res.status(500).send(err);
      });
  },

  update: (req, res) => {
    req.body.updatedAt = new Date().toLocaleDateString();
    User.findByIdAndUpdate(req.params.id, req.body, (err, post) => {
      if (req.body.update === true) {
        post.changePassword(req.body.oldPassword, req.body.newPassword, () => {
          post.save();
        });
      }
      if (err) {
        res.send(err);
        console.log(err);
      }
      res.json(post);
    });
  },

  getOne: (req, res, next) => {
    User.findById(req.params.id)
      .then((userFound) => {
        if (!userFound) {
          return res.status(404).end();
        }
        return res.status(200).json(userFound);
      })
      .catch(err => next(err));
  },
};

```

## user.route.js
> A user route a felhasználókért felel. A *get, put, post és delete* műveleteket a user controllerében definiáltuk.

> A loggedInUser függvény megvizsgálja, hogy van-e báárki is bejelentkezve.
> A loggedIn füügvvény pedig, azt vizsgálja, adminként van-e bejelentkezve.
> Ezzel a módszerrel szabjuk meg ki és mit használhat a CRUD műveletek közül. 

```javascript
const passport = require('passport');
const userRouter = require('express').Router();
const UserController = require('../controller/user.controller');

function loggedIn(req, res, next) {
  if (req.user) {
    if (req.user.rights === true) {
      console.log(req.user);
      next();
    } else {
      res.status(500).json({
        error: 'Wrong Rights!',
      });
    }
  } else {
    res.status(500).json({
      error: 'Access Denied!',
    });
  }
}

function loggedInUser(req, res, next) {
  if (req.user) {
    next();
  } else {
    res.status(500).json({
      error: 'Access Denied!',
    });
  }
}

userRouter.get('/profile', loggedInUser, UserController.profile);
userRouter.get('/listAll', UserController.listAll);
userRouter.get('/getOne/:id', loggedIn, UserController.getOne);
userRouter.delete('/remove/:id', loggedIn, UserController.remove);
userRouter.post('/register', UserController.register);
userRouter.put('/update/:id', loggedInUser, UserController.update);

userRouter.post('/login', passport.authenticate('local'), UserController.login);
userRouter.get('/logout', UserController.logout);

module.exports = userRouter;
```


## mail.route.js
> A nodemailer package segítségével, könnyedén küldhet e-mailben feedbacket a felhasználó.
> Külön route-ban kiszervezve, a kód nem rondítja a server.js-t.

> Biztonsági okokból az e-mail fiók jelszavát cenzúráztuk.
> Mint látható, gmail fiókot használunk.

```javascript
const nodeMailer = require('nodemailer');
const mailRouter = require('express').Router();

mailRouter.post('/send', (req, res, next) => {
  const transporter = nodeMailer.createTransport({
    service: 'gmail',
    host: 'smtp.gmail.com',
    port: 465,
    secure: true,
    auth: {
      user: 'szeszpress@gmail.com',
      pass: '************',
    }
  });
  const mailOptions = {
    from: req.body.from,
    to: 'szeszpress@gmail.com',
    subject: req.body.subject,
    text: req.body.text,
    html: `<p>${req.body.text}
      <br>
      <strong>From: ${req.body.from}</strong>
      </p>`,
  };
  transporter.sendMail(mailOptions, (error, info) => {
    if (error) {
      return console.log(error);
    }
    console.log('Message %s sent: %s', info.messageId, info.response);
    res.render('index');
  });
});

module.exports = mailRouter;
```