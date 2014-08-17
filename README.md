EmailPK - Instant encrypted email
========================

**WARNING: THIS IS A PROOF OF CONCEPT AND SHOULD NOT BE USED FOR REAL SECRETS.**

Demo: https://diafygi.github.io/emailpk/

EmailPK is a public-key encryption website and library that lets users encrypt
easily encrypt and send email message through a variety of webmail interfaces.
A user's email is used as their public key, and their private key is derived
from their passphrase. No key files are needed to successfuly send and receive
encrypted emails.

This project was heavily inspired by [miniLock](https://github.com/kaepora/miniLock),
which uses the same base encryption libraries as EmailPK. Also, a related
project [myLock](https://github.com/diafygi/myLock) uses a similar approach to
EmailPK to encrypt generic files. Check them out!

##Why use EmailPK?

1. The biggest problem with adoption of PGP has been key management. Most email
users have no idea what encryption keys are, much less the difference between
public and private keys. EmailPK attempts to address this problem by removing
the need for key files entirely, and instead focusing on using things the user
already uses: emails and passwords. See [How it works](#how-it-works) for more
details.

2. The only software requirements for sending and receiving encrypted emails
is a modern web browser and an email account on a popular webmail provider. No
additional software or plugins are needed. This allows users to just visit the
demo website above to send or receive an email. This can hopefully lower the
barrier to entry for people to start encrypting their communications.

3. The demonstration website above is made to be [unhosted](https://www.unhosted.org/).
It can be saved to your computer (just right click and "Save As") and opened
directly with no need for an internet connection. No external files are required
and no server calls are made beyond the initial website page load (or no calls
at all if you are hosting the file locally). Additionally, only public webmail
links are used to send emails, so no 3rd-party OAuth approval is required.

4. This entire project is only ~1300 lines of code, and the core library is less
than 300 lines of code. It is meant to be self contained, easy to learn, and
easy to audit (I would love to have a security audit donated to the project).

5. Since EmailPK is open source it can be included with other software to allow
easy asymmetric email encryption. I would love to see people create EmailPK
wrappers for more email providers and social network APIs.

6. EmailPK is designed to be used anonymously. No personally identifiable
information is ever requested, and you can generate as many email codes as you
want. You can use EmailPK as disposable encryption by ignoring the passphrase
field during message composition, which let only the recipients to be able to
decrypt the message (NOTE: the downside to this is that you won't be able to
read any replies to that email either).

##How it works

Public key encryption works by using a public key to encrypt a file, that then
can only be decrypted by the person who has the complementary secret key. This
means you can widely publish your public key (on your twitter profile, email
signature, personal website, etc.) and others can then encrypt files that only
you can decrypt.

Normally, public key encryption requires that the user keep a secret key saved
somewhere on your computer. However, EmailPK uses an algorithm that generates a
secret key from your email and passphrase, so that you only need the email and
passphrase to recreate the secret key (see [Drawbacks](#drawbacks)).

Luckily, many popular webmail providers offer the ability to add aliases to
emails (i.e. diafygi+ABC123@gmail.com), which means that you can add your public
key to your email, which can then be automatically stored in others' contact
lists. This basically lets contact lists act as the public key repository for a
user.

Additionally, this demo uses the public compose deeplinks offered by several
webmail providers to allow for easy sending of encrypted messages. By using
these links, EmailPK doesn't need ask for permission to access their APIs. This
removes the need to trust EmailPK with API permissions.

##Drawbacks

1. Unfortunately, since only your email and passphrase are used to derive your
secret key, it means your passphrase has to be strong, so EmailPK requires a
passphrase that is at least 20 characters to encourage users to use stronger
passphrases. We make it harder to create universal hash or rainbow tables of
passphrases by salting them with the user's email, but common phrases such as
song lyrics and movie quotes should not be used. A
[password manager](https://en.wikipedia.org/wiki/List_of_password_managers) is a
great option to use for generating EmailPK passphrases.

2. Since the email contains the public key for a user, the public key part of
the email cannot be chosen by the user. Instead, that part of the email is
generated when the passphrase is entered. This can make it more difficult to
manually type in someone's email, but the reply functionality in EmailPK
partially mitigates the need for that.

3. You need both your email and your passphrase to log back into your account
because your email is used as a salt to generate your secret key. Without the
email, the original secret key cannot be recreated. Luckily, you don't have
worry about keeping your email secret so you can keep it somewhere you can
easily copy/paste from (or it will pre-filled when decrypting a message).

4. Senders and TO/CC users are included in the clear with the message. This is
to allow easy reply/reply-all that is a common use-case in email. BCC recipients
are not included in the encrypted message, so use that field if you do not want
other recipients to see that recipient.

##Technical details

###External libraries
EmailPK includes three external libraries:

* [TweetNaCl.js](https://github.com/dchest/tweetnacl-js/) - Encryption library
* [scrypt-async.js](https://github.com/dchest/scrypt-async-js/) - Hashing libary
* [Base58.js](https://gist.github.com/diafygi/90a3e80ca1c2793220e5/) - Base 58 encoding/decoding library

These libraries are included inline and minified in the EmailPK `index.html`.
The exact commit link from which the minified source was downloaded is included
in the comment above the code.

###Email encryption steps

1. A user selects to compose a message, enters their email, recipient emails, and composes a message.
2. If they want to read replies, they can enter a passphrase (otherwise one is randomly chosen).
3. The passphrase is hashed using [`nacl.hash()`](https://github.com/dchest/tweetnacl-js#naclhashmessage) (uses SHA-512).
4. The secret key is generated from the passphrase hash and the user's email as salt using [`scrypt(salt, hash, 17, 8, 32, 1000, callback)`](https://github.com/dchest/scrypt-async-js/blob/master/README#L21) (uses scrypt).
5. A public key is derived form the secret key using [`nacl.box.keyPair.fromSecretKey()`](https://github.com/dchest/tweetnacl-js#naclboxkeypairfromsecretkeysecretkey) (uses curve25519).
6. If the user's email already has a public key, it is compared with the generated public key.
7. If the user's email doesn't have a public key, the generated public key is added to their email.
8. A 32-byte random file key and nonce are generated using [`nacl.randomBytes()`](https://github.com/dchest/tweetnacl-js#naclrandombyteslength) (uses window.crypto.getRandomValues).
9. The message is symmetrically encrypted with the file key and nonce using [`nacl.secretbox()`](https://github.com/dchest/tweetnacl-js#naclsecretboxmessage-nonce-key) (uses xsalsa20-poly1305).
10. A hash of the sender's full email (with public key) is hashed using [`nacl.hash()`](https://github.com/dchest/tweetnacl-js#naclhashmessage) (uses SHA-512).
11. A header that contains file key, nonce, and sender email hash is encrypted with the public key of the each recipient, a random nonce, and signed with the sender's secret key using [`nacl.box()`](https://github.com/dchest/tweetnacl-js#naclboxmessage-nonce-theirpublickey-mysecretkey) (uses curve25519-xsalsa20-poly1305).
12. The final message is composed by concatting the sender's public key, the list of headers, and the encrypted message.
13. The final message is encoded from a Uint8Array to a Base58 string.
14. The final message is added to a helpful link that also contains the sender, recipients (to and cc, but not bcc), and a random subject.
15. Reply links use the same random subject by default to allow for easy conversation threading in email clients.
16. Public webmail compose deeplinks are generated that contain the helpful link.

###Email decryption steps
1. The user either clicks on a helpful link they receive or manually pastes the link into the Read Existing textarea.
2. The encrypted message is extracted from the link and decoded to a Uint8Array.
3. The user is asked to enter their email and passphrase (if they haven't already). To and cc recipients are provided in a dropdown to allow for easy email selection.
4. The passphrase is hashed using [`nacl.hash()`](https://github.com/dchest/tweetnacl-js#naclhashmessage) (uses SHA-512).
5. The secret key is generated from the passphrase hash and the user's email as salt using [`scrypt(salt, hash, 17, 8, 32, 1000, callback)`](https://github.com/dchest/scrypt-async-js/blob/master/README#L21) (uses scrypt).
6. Each header is attempted to be decrypted using the secret key, the sender's public key, and the header nonce using [`nacl.box.open()`](https://github.com/dchest/tweetnacl-js#naclboxopenbox-nonce-theirpublickey-mysecretkey) (uses curve25519-xsalsa20-poly1305).
7. If a header is successfully decrypted, the message is decrypted with file key and nonce using [`nacl.secretbox.open()`](https://github.com/dchest/tweetnacl-js#naclsecretboxopenbox-nonce-key) (uses xsalsa20-poly1305).
8. The sender's email is compared to the decrypted senderEmailHash to authenticate the sender using [`nacl.hash()`](https://github.com/dchest/tweetnacl-js#naclhashmessage) (uses SHA-512).
9. If the sender is authenticated, the decrypted message is displayed.
10. The sender's email in the helpful link is compared to the sender's public key included in the encrypted message.
11. If the sender email's public from the helpful link matches the sender's public key from the encrypted message, the sender is displayed and the user can reply to the message.
12. If the sender email's public from the helpful link does not match the sender's public key from the encrypted message, a warning is shown that the sender could not be verified.

###Message format

The helpful link (i.e. what the user clicks on) uses the following format:

```
<BaseURL>                (i.e. https://diafygi.github.io/emailpk/ or file:///path/to/index.html)
?from=<senderEmail>      (contains the sender's public key)
&subject=<randomString>  (random subject field, consistent across replies for easy threading)
&to=<recipientEmail>,... (comma separated emails, if any)
&cc=<recipientEmail>,... (comma separated emails, if any)
&msg=<encryptedMessage>  (the encrypted file)
```

The encrypted message (i.e. what's in the `&msg=<encryptedMessage>`) uses the following format:

```
<encryptedMessage> = [
    <senderPublicKey(32 bytes)>,
    <header(160 bytes)>,
    <header(160 bytes)>,
    ...
    <encryptedBody(variable bytes)>
]
```

Headers use the following format:

```
<header(160 bytes)> = [
    <textInfoNonce(24 bytes)>,
    <textInfoEncrypted(136 bytes)> = [
        <textKey(32 bytes)>,
        <textNonce(24 bytes)>,
        <senderEmailHash(64 bytes)>
    ] (encrypted with recipient.publicKey, textInfoNonce, and sender.secretKey (encryption adds 16 bytes)),
]
```

##EmailPK core library API

####`EmailPK.setEmail(email, passphrase)`

Sets salt, public key, and secret key in the EmailPK object. If email does not
contain a public key, the public key is derived from the email and passphrase
and added back to the email (can be retrieved via the `EmailPK.getEmail()`
function). Passphrases must be at least 20 characters.

####`EmailPK.onEmailDone(error)`

Gets called when a email has been set. A successful completion will leave
error undefined. An unsuccessful completion will have an error string.

####`EmailPK.getEmail()`

Will return the email set in the EmailPK object that contains the user's public
key. NOTE: there is not a way to retrieve the secret key in the EmailPK object.

####`EmailPK.decodeEmail(email)`

Determines if an email contains a valid public key or not. Returns an error
string if there is an error. Returns an object if there is not an error. The
object is `{'salt': <String>}` if the email does not contain a public key. The
object is `{'salt': <String>, 'publicKey': <Uint8Array>}` if the email does
contain a public key. The salt field is the email without the public key.

####`EmailPK.encrypt(text, recipients)`

Encrypt some text for a list of recipients (text is a string, recipients is an
array of emails that contain).

####`EmailPK.onEncryptDone(encrypted_text, error)`

Gets called when the text has been encrypted. A successful completion will leave
error undefined. An unsuccessful completion will have an error string. The
encrypted_text is a base 58 encoded string.

####`EmailPK.decrypt(encrypted_text, sender_email)`

Decrypt an encrypted_text string from a sender's email.

####`EmailPK.onDecryptDone(text, error)`

Gets called when the text has been decrypted. A successful completion will leave
error undefined. An unsuccessful completion will have an error string. The text
variable is the decrypted string.

##Example

```javascript
var EPK = new EmailPK();

//fires on email creation or login
EPK.onEmailDone = function(error){

    //errors are strings or undefined
    if(error !== undefined){
        console.log("Email error: " + error);
        return;
    }

    //get the created email
    var my_email = EPK.getEmail();
    console.log("Email: " + my_email);

    //fires on encrypt completion
    EPK.onEncryptDone = function(enc_txt, error){

        //errors are strings or undefined
        if(error !== undefined){
            console.log("Encryption error: " + error);
            return;
        }

        console.log("Encrypted message: " + enc_txt);

        //fires on decrypt completion
        EPK.onDecryptDone = function(text, error){
            //errors are strings or undefined
            if(error !== undefined){
                console.log("Decryption error: " + error);
                return;
            }

            console.log("Decrypted message from '" + my_email + "': " + text);
        }

        //decrypt the text
        EPK.decrypt(enc_txt, my_email);
    }
    EPK.encrypt("Hello World!", [my_email]);
}
EPK.setEmail("test@example.com", "aaaaaaaaaaaaaaaaaaaa");
```

##Demo

https://diafygi.github.io/emailpk/

##License and Feedback

This project is released under the GPLv2 license, but external libraries may be
licensed differently. This project is hosted on [Github](https://www.github.com/diafygi/emailpk),
so please file bug reports and pull requests there.

