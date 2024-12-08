#show link: underline
#show raw: set block(fill: luma(230), inset: 8pt, radius: 4pt)
#show raw: fragment => {
    show raw.line: it => {
        let symbol = it.text.at(0, default: "")
        let clr = if symbol == "+" {
            rgb("#ccffcc")
        } else if symbol == "-" {
            rgb("#ffcccc")
        }
        if clr != none {
            highlight(fill: clr, it)
        } else {
            it
        }
    }
    fragment
}
#set raw(syntaxes: "../../php-src.sublime-syntax")

#set text(font: "Inter")

= Installing phpMyAdmin

If you have not installed phpMyAdmin yet, you can follow this little guide.
We will set up phpMyAdmin on its own domain.

== Prerequisites

For the sake of this tutorial, we will assume that NGINX and Certbot are already configured and installed. If this is not the case, here are some tutorials that may help you from other sources:

- #link("https://www.nginx.com/resources/wiki/start/topics/tutorials/install/", "NGINX Installation")
- #link("https://certbot.eff.org/", "Certbot Installation")

== Obtaining a Certificate for your phpMyAdmin domain

First of all, we are going to make sure NGINX is off while we obtain the certificate. We are going to do this by executing this command:

```shell
systemctl stop nginx
```

Second of all, we are going to obtain the certbot certificate (replace `phpmyadmin.example.org` with your domain):

```shell
certbot certonly --standalone -d phpmyadmin.example.org
```

== Installing phpMyAdmin

=== Ubuntu and Debian:

```shell
mkdir /var/www/phpmyadmin

cd /var/www/phpmyadmin

wget https://files.phpmyadmin.net/phpMyAdmin/5.2.1/phpMyAdmin-5.2.1-all-languages.zip

unzip phpMyAdmin-5.2.1-all-languages.zip

cd phpMyAdmin-5.2.1-all-languages

mv * /var/www/phpmyadmin

cd /var/www/phpmyadmin

rm -rf phpMyAdmin-5.2.0-all-languages
```

Now that we have installed phpMyAdmin we are going to setup the phpMyAdmin nginx configuration file. Execute these commands:

```bash
nano /etc/nginx/sites-available/phpmyadmin.conf
```

Then copy and paste the code below, but please replace the <domain> sections with your domain e.g. `phpmyadmin.example.org`

```nginx
server {
  listen 80;
  server_name <domain>;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl http2;
  server_name <domain>;

  root /var/www/phpmyadmin;
  index index.php;

  location / {
    try_files $uri $uri/ =404;
  }

  location ~ .php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.4-fpm.sock;
  }

  ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
  ssl_session_cache shared:SSL:10m;
  ssl_protocols TLSv1.2 TLSv1.3;
}
```

*⚠️ Warning ⚠️*

This configuration will work if you are using PHP 8.4, replace the line `fastcgi_pass` with the PHP version installed on your system.

Once we have done that we are going to execute the following commands to finish the installation!

```bash
ln -s /etc/nginx/sites-available/phpmyadmin.conf /etc/nginx/sites-enabled/phpmyadmin.conf
```

```bash
systemctl restart nginx
```

phpMyAdmin is now fully configured. You can try using it at the domain you have set! Please make sure to remember to use `https` instead of `http`!

= Pterodactyl

Move all the files in the `pterodactyl` directory from this archive to your Pterodactyl directory (e.g. `/var/www/pterodactyl`).

Note: You can upload these files even if you have other add-ons or themes installed, it will not affect them.

== Back-end files

=== `app/Providers/RepositoryServiceProvider.php`

Add the following lines after `use Pterodactyl\Contracts\Repository\ServerVariableRepositoryInterface;` (line 39):

```php-src
use Pterodactyl\Contracts\Repository\AktiCubeDevelopmentTeamAutomaticPhpMyAdminRepositoryInterface;
use Pterodactyl\Repositories\Eloquent\AktiCubeDevelopmentTeamAutomaticPhpMyAdminRepository;
```

Then, after `$this->app->bind(UserRepositoryInterface::class, UserRepository::class);` (line 64) add these lines:

```php-src
        $this->app->bind(AktiCubeDevelopmentTeamAutomaticPhpMyAdminRepositoryInterface::class, AktiCubeDevelopmentTeamAutomaticPhpMyAdminRepository::class);
```

=== `app/Models/Permission.php`

Add this line under `public const ACTION_DATABASE_VIEW_PASSWORD = 'database.view_password';` (line 28):

```php-src
    public const ACTION_DATABASE_VIEW_ON_PHPMYADMIN = 'database.view_on_phpmyadmin';
```

Then, after `'view_password' => 'Allows a user to view the password associated with a database instance for this server.',` (line 194) add this line:

```php-src
                'view_on_phpmyadmin' => 'Allows a user to view the database on phpMyAdmin. This permission also requires the VIEW_PASSWORD, READ and UPDATE permission.',
```

=== `app/Services/Databases/DatabaseManagementService.php`

At the end of the file, before `}` and under:
```php-src
        if ($exists) {
            throw new DuplicateDatabaseNameException('A database with that name already exists for this server.');
        }

        $database = (new Database())->forceFill($data);
        $database->saveOrFail();

        return $database;
    }
```

(line 209), add these lines:

```php-src
    public function getPasswordFromDatabase(Database $database): string
    {
        return $this->encrypter->decrypt($database->password);
    }
```

=== `app/Http/Controllers/Api/Client/Servers/DatabaseController.php`

Add these lines after `use Pterodactyl\Http\Requests\Api\Client\Servers\Databases\RotatePasswordRequest;` (line 20):

```php-src
use Illuminate\Http\Resources\Json\JsonResource;
use Pterodactyl\Models\DatabaseHost;
use Pterodactyl\Models\AutomaticPhpMyAdmin;
use Pterodactyl\Http\Requests\Api\Client\Servers\Databases\GetTokenDatabaseRequest;
```

Then, at the end of the files, before `}` and under:
```php-src
        Activity::event('server:database.delete')
            ->subject($database)
            ->property('name', $database->database)
            ->log();

        return Response::create('', Response::HTTP_NO_CONTENT);
    }
```
 (line 132) add these lines:

```php-src
    /**
     * Create a token to use for connection with PhpMyAdmin.
     */
    public function getToken(GetTokenDatabaseRequest $request, Server $server, Database $database): JsonResource
    {
        try {
            $automatic_pma = AutomaticPhpMyAdmin::query()->where('linked_database_host', $database->database_host_id)->firstOrFail();
        } catch (\Exception $e) {
            $automatic_pma = AutomaticPhpMyAdmin::query()->where('linked_database_host', null)->firstOrFail();
        }

        if ($automatic_pma->linked_database_host == null) {
            $database_hosts_ids = DatabaseHost::all()->pluck('id')->toArray();
            $phpmyadmin_server_id = $automatic_pma->phpmyadmin_server_id + array_search($database->database_host_id, $database_hosts_ids);
        } else {
            $phpmyadmin_server_id = $automatic_pma->phpmyadmin_server_id;
        }

        $data = array(
            'phpmyadmin_server_id' => $phpmyadmin_server_id,
            'database_username' => $database->username,
            'database_password' => $this->managementService->getPasswordFromDatabase($database),
        );
        $encryption = openssl_encrypt(json_encode($data), "AES-128-CTR", $automatic_pma->encryption_key, 0, $automatic_pma->encryption_iv);

        return new JsonResource(['encryption' => $encryption, 'url' => $automatic_pma->url, 'cookie_domain' => $automatic_pma->cookie_domain, 'cookie_name' => $automatic_pma->cookie_name]);
    }
```

=== `routes/admin.php`

Add these lines at the end of the file:

```php-src
/*
|--------------------------------------------------------------------------
| Automatic phpMyAdmin Routes
|--------------------------------------------------------------------------
|
| Endpoint: /admin/automatic-phpmyadmin/
|
*/
Route::group(['prefix' => 'automatic-phpmyadmin'], function () {
   Route::get('/', [Admin\AutomaticPhpMyAdminController::class, 'index'])->name('admin.automatic-phpmyadmin');
   Route::get('/new', [Admin\AutomaticPhpMyAdminController::class, 'create'])->name('admin.automatic-phpmyadmin.new');
   Route::get('/view/{automaticphpmyadmin:id}', [Admin\AutomaticPhpMyAdminController::class, 'view'])->name('admin.automatic-phpmyadmin.view');

   Route::post('/new', [Admin\AutomaticPhpMyAdminController::class, 'store']);

   Route::patch('/view/{automaticphpmyadmin:id}', [Admin\AutomaticPhpMyAdminController::class, 'update']);

   Route::delete('/delete/{automaticphpmyadmin:id}', [Admin\AutomaticPhpMyAdminController::class, 'destroy'])->name('admin.automatic-phpmyadmin.delete');
});
```

=== `routes/api-client.php`

After `Route::delete('/{database}', [Client\Servers\DatabaseController::class, 'delete']);` (line 75) add this one:

```php-src
        Route::post('/{database}/getToken', [Client\Servers\DatabaseController::class, 'getToken']);
```

=== `resources/views/layouts/admin.blade.php`

Under:
```blade
                        <li class="{{ ! starts_with(Route::currentRouteName(), 'admin.databases') ?: 'active' }}">
                            <a href="{{ route('admin.databases') }}">
                                <i class="fa fa-database"></i> <span>Databases</span>
                            </a>
                        </li>
```
(line 93) add these:
```blade
                        <li class="{{ ! starts_with(Route::currentRouteName(), 'admin.automatic-phpmyadmin') ?: 'active' }}">
                            <a href="{{ route('admin.automatic-phpmyadmin') }}">
                                <i class="fa fa-database"></i> <span>Automatic phpMyAdmin</span>
                            </a>
                        </li>
```

=== Finishing up

Run these commands:
```bash
cd /var/www/pterodactyl
php artisan migrate # type yes when asked
php artisan optimize
```

You can now access the addon in the admin panel.

= Installing phpMyAdmin side
Like Pterodactyl side, upload all the files of the folder `phpmyadmin` of the ZIP that you have downloaded in the folder of your phpMyAdmin installation.
Then, you will need to modify configuration for the host in `config.inc.php` by adding these lines: 
```php-src
$cfg['Servers'][$i]['auth_type']     = 'signon';
$cfg['Servers'][$i]['SignonSession'] = 'TokenSession';
$cfg['Servers'][$i]['SignonURL']     = 'token.php';
$cfg['Servers'][$i]['LogoutURL']     = 'token-logout.php';
```

A server configuration should now look like this for example:
```php-src
$i++;
$cfg['Servers'][$i]['verbose']       = 'example'; 
$cfg['Servers'][$i]['connect_type']  = 'tcp';
$cfg['Servers'][$i]['host']          = 'localhost';
$cfg['Servers'][$i]['compress']      = false;
$cfg['Servers'][$i]['AllowRoot']     = false;
$cfg['Servers'][$i]['ForceSSL']      = true;
$cfg['Servers'][$i]['auth_type']     = 'signon';
$cfg['Servers'][$i]['SignonSession'] = 'TokenSession';
$cfg['Servers'][$i]['SignonURL']     = 'token.php';
$cfg['Servers'][$i]['LogoutURL']     = 'token-logout.php';
```

You can now edit the file `token.php` that you just uploaded, to fit your needs : 

1. `$encryption_key` (line 14) and `$encryption_iv` (line 16) : These variables must have the same values set in the Pterodactyl side (More information on how to generate these below).
2. `$cookie_name`                                   (line 18) : The name of the cookie to get data from, must be the same as in Pterodactyl side.
3. `$cookie_domain`                                 (line 20) : The domain of the cookie, like the cookie name, must be the same as in Pterodactyl side (More information on how to choose one below).

== Configuration

Go to the admin panel of your Pterodactyl Installation, then select "Automatic phpMyAdmin" in the navigation sidebar, click on create a new installation.
Choose a name and a description.

Then, for the URL, it needs to be the one of your phpMyAdmin installation. You need to include the protocol (e.g. `https://phpmyadmin.example.org`)!

For the cookie name, it needs to be the same as configured in the `token.php` file of your phpMyAdmin installation, same for the `cookie_domain`, the `encryption_key` and the `encryption_iv`. 

When the selected Database Host is none, this phpMyAdmin installation will be used for every Database Host when clicking on the button to connect to the database. But when there is one selected, it'll be this one that will be used when connecting to a database of this host.

Finally, for the phpMyAdmin Server ID, in most cases it'll need to be set to `1`. But if you are using a normal connection on phpMyAdmin like said below, you'll need to change that number to the corresponding configuration in phpMyAdmin. You can now save this installation!

#pagebreak()

=== Q: How can I connect with the default authentication interface of phpMyAdmin alongside the automatic connection from Pterodactyl?
With the use of `signon` for the authentication mode, it is impossible to put in addition the `cookie` mode which is the one which allows the usual connection. To solve this problem, you have to make a new connection host but not specify the authentication mode. 

Example: 
```php-src
$i = 0;

/**
 * First server used for normal connection.
 */
$i++; // <- We increment of 1 the $i value, so this configuration as the ID of 1.
$cfg['Servers'][$i]['verbose'] = 'example-database'; 
$cfg['Servers'][$i]['host'] = 'localhost';

/**
 * Second server used for connection with Pterodactyl
 */
$i++; // <- We increment again the $i value of 1, so this configuration as the ID of 2.
$cfg['Servers'][$i]['verbose'] = 'example-database (Connection with Pterodactyl)'; 
$cfg['Servers'][$i]['host'] = 'localhost';
$cfg['Servers'][$i]['auth_type']     = 'signon';
$cfg['Servers'][$i]['SignonSession'] = 'TokenSession';
$cfg['Servers'][$i]['SignonURL']     = 'token.php';
$cfg['Servers'][$i]['LogoutURL']     = 'token-logout.php';
```
Now you can also connect to your database with the default phpMyAdmin login form.

In this case, the phpMyAdmin Server ID field in Pterodactyl needs to be set to 2, since the ID for the connection with Pterodactyl is also 2.

=== Q: Why should I put a `.` at the beginning of my domain name when my Pterodactyl and phpMyAdmin installation are on a different subdomain?
Let's say your Pterodactyl installation is on the subdomain `panel.example.xyz`, and your phpMyAdmin installation is on the subdomain `database.example.xyz`. You'll need to use `.example.xyz` as the cookie domain in Pterodactyl and phpMyAdmin. Like this, the addon will be able to work on your two installations on two different subdomains.

=== Q: How to generate an Encryption Key and IV?
Those are only really useful on HTTP without any TLS certificate, but who doesn't like some extra encryption? To generate them, you can use a password generator website like #link("https://www.lastpass.com/features/password-generator", "LastPass"). For the Encryption Key, it can be as long as you want with any characters. But the Encryption IV *needs* to be made of exactly 16 digits.

The addon is now fully configured. Go to the "Databases" tab of any of your server with an active database and click the button to access your data right away!

== Front-end files

=== `resources/scripts/components/server/databases/DatabaseRow.tsx`

Under `import CopyOnClick from '@/components/elements/CopyOnClick';` (line 21) add:

`import getTokenDatabase from "@/api/server/databases/getTokenDatabase";`

Then, under `const removeDatabase = ServerContext.useStoreActions((actions) => actions.databases.removeDatabase);`  (line 37) add:

```tsx
    const openpmaURL = () => {
        getTokenDatabase(uuid, database.id)
            .then(data => {
                if(data){
                    const now = new Date();
                    now.setTime(now.getTime()+(2*60*1000));
                    document.cookie = data['cookie_name'] +"=" + data['encryption'] + ";expires=" + now.toUTCString() + ";domain=" + data['cookie_domain'] + ";path=/";
                    const newWindow = window.open(data['url'], '_blank', 'noopener,noreferrer')
                    if (newWindow) newWindow.opener = null
                }
            })
            .catch(error => {
                console.error(error);
            });
    }
```

Finally, under:
```tsx
                    <Button isSecondary css={tw`mr-2`} onClick={() => setConnectionVisible(true)}>
                        <FontAwesomeIcon icon={faEye} fixedWidth />
                    </Button>
```

(line 188) add:

```tsx
                    {
                        database.allowConnectionsFrom === '%' ? (
                            <Can action={'database.view_on_phpmyadmin'}>
                                <Button css={tw`mr-2`} isSecondary onClick={openpmaURL}>
                                    <FontAwesomeIcon icon={faDatabase} fixedWidth/>
                                </Button>
                            </Can>
                        ) : null
                    }
```

If you're going to use a Node.js version greater than or equal to 17.x, you may encounter errors due to an incompatibility between the new OpenSSL and the version of Webpack used by Pterodactyl. As a workaround, set this environment variable before building assets:

```bash
export NODE_OPTIONS=--openssl-legacy-provider
```

Run these commands to build the front-end assets:

```bash
cd /var/www/pterodactyl
yarn install
yarn build:production # can take a while depending on your server
```
