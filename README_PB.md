## user.controller.js

> A user.controller.js segítségével, felhasználókat tudunk létrehozni, szerkeszteni, törölni, vagy keresni.

> **profile**: a bejelentkezett felhasználó, saját adatait adja vissza.
> 
> **listAll**: összes regisztrált felhasználó listázása.
> 
> **register**: felhasználó regisztrálása név, e-mail és jogosultsággal.
> 
> **update**: a felhasználó adatainak frissítése. A *changePassword* metódusa a passport-local-mongoose egyik beépített metódusa, mely segítségével, frissíthetjük a felhasználó jelszavát.
> 
> **getOne**: adott id-jú felhasználó keresése. Ha nem találja a megadott felhasználót, akkor 404-es hibakóddal tér vissza, ha pedig sikeresen megtalálta, 200-as kóddal, és a felhasználó adataival.
> 
> **remove**: adott id-jú felhasználó törlése.

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
> A *user.route* a felhasználókért felel. A *get, put, post és delete* műveleteket a user controllerében definiáltuk.

> A loggedInUser függvény megvizsgálja, hogy van-e bárki is bejelentkezve.
> 
> A loggedIn füügvény pedig, azt vizsgálja, milyen jogosultsággal van bejelentkezve a felhasználó. Ha a jogosultsága **true**, akkor a felhasználó admin, és joga van bármely regisztrált felhasználón módosításokat végezni, úgy mint: *kilistázni az összeset, törölni őket, más jogosultságot adni nekik, vagy létrehozni újakat*.
> 
> Ha a felhasználó nem admin jogosultságú, akkor csak a saját profilját érheti el, és módosíthatja.

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
        error: 'Nem megfelelő jogosultság!',
      });
    }
  } else {
    res.status(500).json({
      error: 'Belépés megtagadva!',
    });
  }
}

function loggedInUser(req, res, next) {
  if (req.user) {
    next();
  } else {
    res.status(500).json({
      error: 'Belépés megtagadva!',
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
> A webshpunkon levő *Kapcsolat* oldalon, nodemailer package segítségével, könnyedén küldhet a felhasználó visszajelzést nekünk.
> 
> Külön route-ban kiszervezve, a kód nem rondítja a server.js-t, és jól átlátható marad.

> A **nodeMailer** beépített metódusában, a *createTransport*-ben megadtuk a szolgáltó nevét, host címét és port számát, illetve a saját e-mail fiókunk adatait, hogy ezen keresztül legyen elérhető a felhasználó üzenete számunkra.
> 
> Mint látható, gmail fiókot használunk.
> 
> A *mailOptions*-ben eltároljuk az e-mail minden fontos elemét. Például, hogy kitől jött az üzenet, az üzenet tárgyát, és persze az üzenet tartalmát.



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
      pass: 'szeszpress2018',
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