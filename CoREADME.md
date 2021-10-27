# mangal-app
NGSoftServices


User Model...
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

//Create Schema
const UserSchema = new Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true
  },
  password: {
    type: String,
    required: true
  },
  avatar: {
    type: String
  },
  date: {
    type: Date,
    default: Date.now
  }
});

module.exports = User = mongoose.model("users", UserSchema);


UserRouter.....
const express = require("express");
const gravatar = require("gravatar");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const passport = require("passport");
const AUTHSECRET = require("../../config/keys").authSecret;

const router = express.Router();

//Load User Validations
const validateRegisterInput = require("../../validations/register");
const validateLoginInput = require("../../validations/login");

//Load User Model
const User = require("../../models/User");

//@route    GET/api/users/test
//@desc     Test User Routes
//@access   Public
router.get("/test", (req, res) => {
  res.json({
    msg: "Users Works"
  });
});

//@route    GET/api/users/register
//@desc     Register Users
//@access   Public

router.post("/register", (req, res) => {
  const { errors, isValid } = validateRegisterInput(req.body);
  if (!isValid) {
    return res.status(400).json({ errors });
  }

  User.findOne({ email: req.body.email }).then(user => {
    if (user) {
      errors.email = "Email Already Exists";
      return res.status(400).json({ errors });
    } else {
      const avatar = gravatar.url(req.body.email, {
        s: "200",
        r: "pg",
        d: "mm"
      });

      const newUser = new User({
        name: req.body.name,
        email: req.body.email,
        avatar,
        password: req.body.password
      });

      bcrypt.genSalt(10, (err, salt) => {
        if (err) throw err;
        bcrypt.hash(newUser.password, salt, (err, hash) => {
          if (err) throw err;
          newUser.password = hash;
          newUser
            .save()
            .then(user => res.json(user))
            .catch(err => console.log(err));
        });
      });
    }
  });
});

//@route    GET/api/users/login
//@desc     Login Users
//@access   Public

router.post("/login", (req, res) => {
  const { errors, isValid } = validateLoginInput(req.body);
  console.log(isValid);
  if (!isValid) {
    return res.status(400).json({ errors });
  }

  const email = req.body.email;
  const password = req.body.password;

  User.findOne({ email }).then(user => {
    if (!user) {
      errors.email = "User not found";
      res.status(404).json({ errors });
    }

    bcrypt.compare(password, user.password).then(isMatch => {
      //returns a bool values in isMatch
      if (isMatch) {
        //User matched
        //res.json({ msg: "Success" });
        const payload = {
          id: user.id,
          name: user.name,
          avatar: user.avatar
        };

        jwt.sign(payload, AUTHSECRET, { expiresIn: 3600 }, (err, token) => {
          //
          res.json({
            success: true,
            token: "Bearer " + token
          });
        });
      } else {
        //Incorrect Password
        errors.password = "Invalid Login Credentials";
        return res.status(400).json({ errors });
      }
    });
  });
});

//@route    GET/api/users/current
//@desc     Gets the Current User
//@access   Private

router.get(
  "/current",
  passport.authenticate("jwt", { session: false }),
  (req, res) => {
    res.json({
      id: req.user.id,
      email: req.user.email,
      name: req.user.name
    });
  }
);

module.exports = router;


IS-emptyfil-check...
const isEmpty = value =>
  value === undefined ||
  value === null ||
  (typeof value === "object" && Object.keys(value).length === 0) ||
  (typeof value === "string" && value.trim().length === 0);

module.exports = isEmpty;


register validation...
const Validator = require("validator");
const isEmpty = require("./is-empty");

module.exports = function validateRegisterInput(data) {
  let errors = {};

  data.name = !isEmpty(data.name) ? data.name : "";
  data.email = !isEmpty(data.email) ? data.email : "";
  data.password = !isEmpty(data.password) ? data.password : "";
  data.password2 = !isEmpty(data.password2) ? data.password2 : "";

  if (!Validator.isLength(data.name, { min: 2, max: 30 })) {
    errors.name = "Name must be between 2 and 30 characters";
  }

  if (Validator.isEmpty(data.name)) {
    errors.name = "Name field is required";
  }

  if (!Validator.isEmail(data.email)) {
    errors.email = "Email is invalid";
  }

  if (Validator.isEmpty(data.email)) {
    errors.email = "Email field is required";
  }

  if (Validator.isEmpty(data.password)) {
    errors.password = "Password field is required";
  }
  if (Validator.isEmpty(data.password2)) {
    errors.password2 = "Confirm Password field is required";
  }

  if (!Validator.equals(data.password, data.password2)) {
    errors.match = "Passwords do not match";
  }

  if (!Validator.isLength(data.password, { min: 6, max: 30 })) {
    errors.password = "Password must be between 6 to 30 characters";
  }

  return {
    errors,
    isValid: isEmpty(errors)
  };
};


is Login Validation...
const Validator = require("validator");
const isEmpty = require("./is-empty");

module.exports = function validateLoginInput(data) {
  let errors = {};

  data.email = !isEmpty(data.email) ? data.email : "";
  data.password = !isEmpty(data.password) ? data.password : "";

  if (!Validator.isEmail(data.email)) {
    errors.email = "Email is invalid";
  }

  if (Validator.isEmpty(data.email)) {
    errors.email = "Email field is required";
  }

  if (Validator.isEmpty(data.password)) {
    errors.password = "Password field is required";
  }

  return {
    errors,
    isValid: isEmpty(errors)
  };
};


Indexserver.....

const express = require('express');
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const passport = require('passport');

const users = require('./routes/api/users');
const profile = require('./routes/api/profile');
const posts = require('./routes/api/posts');

const app = express();

//Body Parser
app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json());

//MongoDB Config
const dbConfig = require('./config/keys').mongoUri;

//Connect to MongoDB
mongoose
  .connect(dbConfig)
  .then(() => console.log('MongoDB Connected'))
  .catch(() => console.log('Error Connecting to MongoDB'));

//Passport Middleware
app.use(passport.initialize());

//Passport Config
require('./config/passport')(passport);

//Use Routes

app.use('/api/users', users);
app.use('/api/profile', profile);
app.use('/api/posts', posts);

const port = process.env.PORT || 5000;

app.listen(port, () => {
  console.log(`Server Running on port ${port}`);
});

config/passport.js....\
const JwtStrategy = require("passport-jwt").Strategy;
const ExtractJwt = require("passport-jwt").ExtractJwt;
const mongoose = require("mongoose");
const User = mongoose.model("users");
const keys = require("./keys");

const opts = {};
opts.jwtFromRequest = ExtractJwt.fromAuthHeaderAsBearerToken();
opts.secretOrKey = keys.authSecret;

module.exports = passport => {
  passport.use(
    new JwtStrategy(opts, (jwtPayload, done) => {
      User.findById(jwtPayload.id)
        .then(user => {
          if (user) {
            return done(null, user);
          }
          return done(null, false);
        })
        .catch(err => console.log(err));
    })
  );
};
