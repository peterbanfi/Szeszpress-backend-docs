
### mail.route.js
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