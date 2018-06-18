## categories.controller.js

```javascript
const Categories = require('../models/categories');
const mongoose = require('mongoose');
mongoose.Promise = require('bluebird');
```

> lowerCaser: A kapott String típusú adatok formázásaa kisbetűssé

```javascript
function lowerCaser(body) {
  const newBody = body;
  Object.keys(newBody).forEach((key) => {
    if (typeof newBody[key] === 'string') {
      newBody[key] = newBody[key].toLowerCase();
    }
  });
  return newBody;
}
```

> list: Az összes kategória listázása

```javascript
  list: (req, res) => {
    Categories.find({})
      .then((categories) => {
        res.status(200).json(categories);
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },
```

> find: Egy kategória megjelenítése

```javascript
  find: (req, res) => {
    Categories.findById(req.params.id)
      .then((categories) => {
        if (categories) {
          res.status(200).json(categories);
        } else {
          res.status(404).json({
            message: 'Nem létező Id!',
          });
        }
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },
```

> create: Egy új kategória létrehozása
 
```javascript
  create: (req, res) => {
    let body = JSON.stringify(req.body);
    body = JSON.parse(body);

    lowerCaser(body);

    Categories.create(body)
      .then((categories) => {
        res.status(200).json(categories);
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },
```

> update: Egy kategória nevének frissítése

```javascript
  update: (req, res) => {
    let body = JSON.stringify(req.body);
    body = JSON.parse(body);

    lowerCaser(body);

    Categories.findById(req.params.id)
      .then(Categories.findByIdAndUpdate(req.params.id, body)
        .then((categories) => {
          if (categories) {
            res.status(200).json(categories);
          } else {
            res.status(404).json({
              message: 'Nem létező Id!',
            });
          }
        })
        .catch((err) => {
          res.status(500).json({
            error: err,
          });
        }));
  },
```

> remove: Egy kategória törlése

```javascript
  remove: (req, res) => {
    Categories.findByIdAndRemove(req.params.id)
      .then((categories) => {
        if (categories) {
          res.status(200).json(categories);
        } else {
          res.status(404).json({
            message: 'Not a valid Id!',
          });
        }
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },
```

## products.controller.js

> list: Az összes termék listázása
> find: Egy termék megjelenítése
> create: Egy új termék létrehozása
> update: Egy termék nevének frissítése
> remove: Egy termék törlése
> addComment: Egy komment hozzáadása
> removeComment: Egy komment törlése

> A képek feltöltését a multer csomaggal oldottuk meg. A feltöltést a products.route.js kezeli itt csak megnézzük a create eljárásban, hogy létezik-e feltöltött fájl és ha igen, akkor cask az elérési útvonalát adjuk hozzá a termékhez, mivel szűkös a MLab felhőtárhelyünk.

> Hasonlóképpen teszünk az update-nél is, ha létezik új kép akkor felülírjuk az elérési útvonalat. Viszont ekkor a már létező régi képet is (ha létezik ilyen) töröljük. Ha nem kapunk új fájlt, akkor megtartjuk a régi elérési útvonalat és képet. Ezt manuálisan kell visszarakni mielőtt továbbengednénk a mongoDb parancsot, mivel ez a PUT request-nél nálunk csak annyit jelent, hogy frontenden nem kívántak képet változtani.

> A remove is leellenőrzi, hogy létezik-e feltöltött kép a termék törlése előtt, ha igen akkor töröljük azt is az /uploads mappánkból.

> Add comment Frédi írja

> Remove comment Frédi írja

```javascript
const Products = require('../models/products');
const mongoose = require('mongoose');
const fs = require('fs');
mongoose.Promise = require('bluebird');

module.exports = {
  list: (req, res) => {
    Products.find({})
      .populate('productCategory', 'categoryName')
      .exec()
      .then((products) => {
        res.status(200).json(products);
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },

  find: (req, res) => {
    Products.findById(req.params.id)
      .populate('productCategory', 'categoryName')
      .populate({
        path: 'productComments',
        select: 'text user createdAt',
        populate: {
          path: 'user',
          select: 'username',
          model: 'User',
        },
      })
      .exec()
      .then((products) => {
        if (products) {
          res.status(200).json(products);
        } else {
          res.status(404).json({
            message: 'Not a valid Id!',
          });
        }
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },

  create: (req, res) => {
    let body = JSON.stringify(req.body);
    body = JSON.parse(body);

    if (req.file) {
      body.productImg = `http://localhost:8080/${req.file.path.replace(/\\/, '/')}`;
    }

    Products.create(body)
      .then((products) => {
        res.status(200).json(products);
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },

  update: (req, res) => {
    let body = JSON.stringify(req.body);
    body = JSON.parse(body);

    if (req.file) {
      body.productImg = `http://localhost:8080/${req.file.path.replace(/\\/, '/')}`;
    }

    if (!body.productImg) {
      Products.findById(req.params.id)
        .then((products) => {
          body.productImg = products.productImg;
        })
        .then(Products.findByIdAndUpdate(req.params.id, body)
          .then((products) => {
            if (products) {
              res.status(200).json(products);
            } else {
              res.status(404).json({
                message: 'Nem létező Id!',
              });
            }
          })
          .catch((err) => {
            res.status(500).json({
              error: err,
            });
          }));
    } else {
      Products.findByIdAndUpdate(req.params.id, body)
        .then((products) => {
          let imgRoute = products.productImg;
          imgRoute = imgRoute.substring(22);

          fs.exists(imgRoute, (exists) => {
            if (exists) {
              fs.unlink(imgRoute, (err) => {
                if (err) throw err;
              });
            }
          });

          if (products) {
            res.status(200).json(products);
          } else {
            res.status(404).json({
              message: 'Not a valid Id!',
            });
          }
        })
        .catch((err) => {
          res.status(500).json({
            error: err,
          });
        });
    }
  },

  remove: (req, res) => {
    Products.findByIdAndRemove(req.params.id)
      .then((products) => {
        if (products) {
          res.status(200).json(products);
        } else {
          res.status(404).json({
            message: 'Nem létező Id!',
          });
        }
        let imgRoute = '';
        if (products.productImg) {
          imgRoute = products.productImg;
          imgRoute = imgRoute.substring(22);
        }

        fs.exists(imgRoute, (exists) => {
          if (exists) {
            fs.unlink(imgRoute, (err) => {
              if (err) throw err;
            });
          }
        });
      })
      .catch((err) => {
        res.status(500).json({
          error: err,
        });
      });
  },

  addComment: (productId, commentId) => Products.findByIdAndUpdate(productId, {
    $push: {
      productComments: commentId,
    },
  }),

  removeComment: (productId, commentId) => Products.findByIdAndUpdate(productId, {
    $pull: {
      productComments: commentId,
    },
  }),
};
```

## server.js
> pam pam

```javascript
const express = require('express');
const mongoose = require('mongoose');
const passport = require('passport');
const morgan = require('morgan');
const bodyParser = require('body-parser');
const session = require('express-session');
const fs = require('fs');
const path = require('path');
const rfs = require('rotating-file-stream');
const helmet = require('helmet');
const LocalStrategy = require('passport-local').Strategy;
const db = require('./config/database.js');
const User = require('./models/user');
const userRouter = require('./route/user.route');
const productsRouter = require('./route/products.route');
const ordersRouter = require('./route/order.route');
const categoriesRouter = require('./route/categories.route');
const commentRouter = require('./route/comment.route');
const mailRouter = require('./route/mail.route');

const logDirectory = path.join(__dirname, 'log');
const port = process.env.PORT || 8080;
const app = express();

// Logging
if (!fs.existsSync(logDirectory)) {
    fs.mkdirSync(logDirectory);
}
const accessLogStream = rfs('access.log', {
    interval: '1d',
    path: logDirectory,
});
app.use(morgan('combined', {
    stream: accessLogStream,
    skip: (req, res) => res.statusCode < 400,
}));

// Security
app.use(helmet());

// Product pictures folder
app.use('/uploads', express.static('uploads'));

// Body Parse middleware
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({
    extended: false,
}));

// Session handling
app.use(session({
    secret: 'secret',
    resave: true,
    httpOnly: false,
    saveUninitialized: true,
    cookie: {
        maxAge: 7 * 24 * 60 * 60 * 1000, // seconds which equals 1 week
    },
}));

// Passport - Auth
app.use(passport.initialize());
app.use(passport.session());
passport.use(new LocalStrategy(User.authenticate()));
passport.serializeUser(User.serializeUser());
passport.deserializeUser(User.deserializeUser());

// Connect to MongoDB
mongoose.connect(db.uri, db.options)
    .then(() => {
        console.log('MongoDB connected.');
    })
    .catch((err) => {
        console.error(`MongoDB error.:${err}`);
    });

// CORS for frontend
app.use((req, res, next) => {
    res.header('Access-Control-Allow-Credentials', true);
    res.header('Access-Control-Allow-Origin', 'http://localhost:4200');
    res.header('Access-Control-Allow-Headers', 'lazyUpdate, normalizedNames, headers, Origin, X-Requested-With, Content-Type, Accept, Authorization');
    if (req.method === 'OPTIONS') {
        res.header('Access-Control-Allow-Methods', 'POST, GET, PUT, PATCH, DELETE');
        return res.status(200).json({});
    }
    return next();
});

// User User router
app.use('/user/', userRouter);
app.use('/products/', productsRouter);
app.use('/orders/', ordersRouter);
app.use('/categories/', categoriesRouter);
app.use('/comments/', commentRouter);
app.use('/', mailRouter);

// 404 error handling
app.use((req, res, next) => {
    const error = new Error('Not Found');
    error.status = 404;
    next(error);
});

// 500 error
app.use((error, req, res) => {
    res.status(error.status || 500);
    res.json({
        error: {
            message: error.message,
        },
    });
});

// Start server
app.listen(port);
```