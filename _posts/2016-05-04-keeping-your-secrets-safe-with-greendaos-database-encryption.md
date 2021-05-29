---
layout: post
title:  "Keeping Your Secrets Safe With greenDAO's Database Encryption"
date:   2016-05-04 00:00:00 -0400
tags: [android, greendao, java]
---
One of the most popular Android ORM solutions out there, greenDAO, has 
[recently been updated][greendao-release] with a new exciting feature - the database encryption. 
greenDAO now supports [SQLCipher][sqlcipher] - an open source SQLite extension, that provides 
transparent 256-bit AES encryption of database files. Data security is an important topic for mobile 
apps, and a number of popular applications have recently announced security-related features - this 
is when a straightforward solution for database encryption comes handy! In this article we'll see 
how easy it is to set up greenDAO with database encryption and start keeping your users' secrets 
safe.

### My Secrets app

For demonstration purposes, we'll be using a simple application called My Secrets. My Secrets does 
just one thing, and does it well, - it stores your secrets in a safe place. It allows you to share 
anything, even the dirtiest little things you've always been afraid to tell anybody:

{:refdef: style="text-align: center;"}
![ZrdahPo](/assets/ZrdahPo.png)
{: refdef}

and stores them safely, so that nobody ever finds out:

{:refdef: style="text-align: center;"}
![bTvzfuy](/assets/bTvzfuy.png)
{: refdef}

We'll encrypt the database file, so in case it gets into the hands of an attacker, they won't be 
able to access the data stored inside. Let's see how to achieve this level of security using 
greenDAO.

### Setting up greenDAO dependencies

The setup is pretty straightforward, we'll start with adding the following dependencies to the app 
module's build file:

```groovy
compile "org.greenrobot:greendao-encryption:$greendao_version"
compile "net.zetetic:android-database-sqlcipher:$sqlcipher_version"
```

and the following to the DAO generator module:

```groovy
compile "org.greenrobot:greendao-generator-encryption:$greendao_version"
```

In case you're not familiar with the typical greenDAO setup, make sure to check the official 
[How to get started][how-to-get-started] guide.

Now let's create the `MySecretsDaoGenerator` class that will generate all necessary DAO-related 
classes:

```java
public class MySecretsDaoGenerator {

    private static final String PACKAGE_NAME = "me.egorand.mysecrets.data.gen";

    public static void main(String[] args) throws Exception {
        Schema schema = new Schema(1, PACKAGE_NAME);

        addSecret(schema);

        new DaoGenerator().generateAll(schema, "../app/src/main/java");
    }

    private static void addSecret(Schema schema) {
        Entity secret = schema.addEntity("Secret");
        secret.addIdProperty();
        secret.addStringProperty("text").notNull();
        secret.addDateProperty("addedDate").notNull();
    }
}
```

Running this class will generate 4 classes for us - `Secret`, `SecretDao`, `DaoSession` and 
`DaoMaster`. We'll use those later.

### Creating the SecretsRepository

`SecretsRepository` is our abstraction over greenDAO, that provides app-specific methods for data 
access. We'll be passing in an instance of `DaoSession`, which is the recommended approach. 
`SecretsRepository` defines the following methods:

```java
@Singleton
public class SecretsRepository {

    private final SecretDao secretDao;

    public SecretsRepository(DaoSession session) {
        this.secretDao = session.getSecretDao();
    }

    public List<Secret> loadAllSecrets() {
        return secretDao.loadAll();
    }

    public void storeSecret(String secretText) {
        Secret secret = new Secret();
        secret.setText(secretText);
        secret.setAddedDate(new Date());
        secretDao.insert(secret);
    }
}
```

And we'll also define an @Provides method inside our Dagger module that will initialize 
`SecretsRepository`:

```java
@Provides @Singleton public SecretsRepository provideSecretsRepository(SecretsDatabaseKeyHolder
                                                                                   secretsDatabaseKeyHolder) {
        EncryptedDevOpenHelper helper = new EncryptedDevOpenHelper(context, "secrets.db");
        Database database = helper.getWritableDatabase(secretsDatabaseKeyHolder.getKey());
        DaoMaster daoMaster = new DaoMaster(database);
        return new SecretsRepository(daoMaster.newSession());
    }
```

We'll be using one of the generated "open helper" implementations that greenDAO provides, the 
`EncryptedDevOpenHelper`. `EncryptedDevOpenHelper` will just drop and recreate all tables in
`onUpgrade()`, which is totally fine in our case; if you'd like a more fine-grained control over 
this behavior - you should go with extending another class called `EncryptedOpenHelper`.

You can see that the access to the database is done via `helper.getWritableDatabase()`. Since the 
database is encrypted, we need to provide a decryption key to initialize the connection. We'll use 
the `SecretsDatabaseKeyHolder` class, which will generate a strong key and will keep it for later 
use. Let's implement the key generation first.

### Generating the database key

There's a pretty old [article][using-cryptography] on the Android Developers Blog that discusses 
solutions for generating strong cryptographic keys. We'll create the following class based on that 
approach:

```java
public class SecretsDatabaseKeyGenerator {

    public static final int DEFAULT_KEY_LENGTH = 256;

    public static String generateKey() {
        try {
            SecretKey secretKey = internalGenerateKey();
            return Base64.encodeToString(secretKey.getEncoded(), Base64.NO_WRAP);
        } catch (NoSuchAlgorithmException ignored) {
        }
        return null;
    }

    private static SecretKey internalGenerateKey() throws NoSuchAlgorithmException {
        SecureRandom random = new SecureRandom();
        KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
        keyGenerator.init(DEFAULT_KEY_LENGTH, random);
        return keyGenerator.generateKey();
    }
}
```

The key generation snippet yields an instance of `SecretKey`, which holds the actual key data within
a `byte[]`. We'll Base64-encode the bytes to receive a `String`, which can be used as a password for 
the encrypted database.

### Storing the database key

Unfortunately, storing app secrets on an Android device is not easy. There are APIs that seem to 
provide somewhat related functionality, like the [KeyChain][keychain], but those aren't very 
developer friendly, and their behavior varies significantly on different Android versions. The 
bottom line is that you should avoid storing any sensitive data unencrypted on an Android device. 
Storing app secrets securely is not in the scope of this article, and probably deserves a separate 
article. We'll just keep it simple:

```java
@Singleton
public class SecretsDatabaseKeyHolder {

    private static final String LOG_TAG = "KeyHolder";

    private static final String PREFS_NAME = "default";

    private static final String KEY_DB_KEY = "db_key";

    private final SharedPreferences prefs;

    public SecretsDatabaseKeyHolder(Context context) {
        this.prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
    }

    public String getKey() {
        String key = prefs.getString(KEY_DB_KEY, null);
        if (key == null) {
            key = SecretsDatabaseKeyGenerator.generateKey();
            prefs.edit().putString(KEY_DB_KEY, key).apply();
        }
        if (BuildConfig.DEBUG) {
            Log.d(LOG_TAG, "DB key: " + key);
        }
        return key;
    }
}
```

As you see, we're just writing the key to the `SharedPreferences`. But I should warn you: 
**Never ever do this in a production app!** Thing is, `SharedPreferences` uses a simple XML file to 
store your data in plain text, which makes it extremely easy for an attacker to access the info. 
While this approach is fine for a demo app, make sure you come up with something more secure for the
apps you publish to your users.

So now we have a way to initialize the key for database encryption and store it for later use. 
That's all we need, so let's finish with writing some code that accesses the database.

### Creating the SecretsLoader

Populating a `RecyclerView` with data is done with the help of a `Loader`:

```java
public class SecretsLoader extends AsyncTaskLoader<List<Secret>> {

    private final SecretsRepository secretsRepository;

    private List<Secret> cachedSecrets;

    @Inject public SecretsLoader(Context context, SecretsRepository secretsRepository) {
        super(context);
        this.secretsRepository = secretsRepository;
    }

    @Override protected void onStartLoading() {
        if (cachedSecrets != null) {
            deliverResult(cachedSecrets);
        } else {
            forceLoad();
        }
    }

    @Override public List<Secret> loadInBackground() {
        return secretsRepository.loadAllSecrets();
    }

    @Override public void deliverResult(List<Secret> data) {
        cachedSecrets = data;
        super.deliverResult(data);
    }
}
```

### Adding secrets from an Activity

The main `Activity` of our application, the `SecretsActivity`, has the following code to create new 
entries inside the database:

```java
@Override public void onKeepNewSecret(String secretText) {
    secretsRepository.storeSecret(secretText);
    getSupportLoaderManager().restartLoader(LOADER_SECRETS, null, this);
}
```

The code creates a new `Secret` entry via the `SecretsRepository` and restarts the loader to refresh 
the UI.

The rest of the application code is available on [GitHub][github].

### Hacking My Secrets

Now, in order to check that we've set up an encrypted database properly, let's try to access it 
using the [SQLite Database Browser][sqlite-browser] app (make sure you're using the version that 
supports SQLCipher) and a [super handy script][script] by Cedric Beust, that helps pull the database
file from the device and open it in the Database Browser with a single command. The script is 
tailored to the needs of our app and is added to the GitHub repo:

```bash
#
# pull-db
# Inspect the database from your device
# Cedric Beust
#

PKG=me.egorand.mysecrets
DB=secrets.db

adb shell "run-as $PKG chmod 755 /data/data/$PKG/databases"
adb shell "run-as $PKG chmod 666 /data/data/$PKG/databases/$DB"
adb shell "rm /sdcard/$DB"
adb shell "cp /data/data/$PKG/databases/$DB /sdcard/$DB"

mkdir tmp
rm -f tmp/${DB}
adb pull /sdcard/${DB} tmp/${DB}

open /Applications/SQLite\ Browser.app tmp/${DB}
```

Let's now run the script and see what happens:

```bash
$ ./pull-db.sh
```

SQLite Database Browser app is started, and we're seeing the following dialog:

{:refdef: style="text-align: center;"}
![STV0X6F](/assets/STV0X6F.png)
{: refdef}

The app won't let us see the data without providing the password - looks good! And what happens if 
we just Cancel and go forward?

{:refdef: style="text-align: center;"}
![U3Mj5p6](/assets/U3Mj5p6.png)
{: refdef}

No way. Let's try it again, but now we'll provide the password, which we're printing into the logcat
for debugging purposes:

{:refdef: style="text-align: center;"}
![TFdJkTL](/assets/TFdJkTL.png)
{: refdef}

Looks a lot better! The Database Browser was able to decrypt the file using the password we've 
provided.

### Conclusion

Securing user data inside mobile applications can be tricky. By default, databases have zero 
encryption on Android, so all the data is easily readable by database browsers. If you feel it's a 
good idea to encrypt databases inside your apps - greenDAO provides an out of the box solution, 
which as we saw is trivial to setup and use. So keep your secrets safe, and happy coding!

Thanks a ton to [Markus Junginger][markus] for the proofreading.

[greendao-release]: https://greenrobot.org/release/greendao-2-2-with-database-encryption/
[sqlcipher]: https://www.zetetic.net/sqlcipher/
[how-to-get-started]: https://greenrobot.org/greendao/documentation/how-to-get-started/
[using-cryptography]: https://android-developers.blogspot.de/2013/02/using-cryptography-to-store-credentials.html
[keychain]: https://developer.android.com/reference/android/security/KeyChain.html
[github]: https://github.com/Egorand/android-greendao-database-encryption
[sqlite-browser]: https://sqlitebrowser.org/
[script]: https://beust.com/weblog/2015/07/09/easily-inspect-your-sqlite-database-on-android/
[markus]: https://twitter.com/greenrobot_de
