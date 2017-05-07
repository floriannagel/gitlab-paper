# Gitlab CI + CodeIgniter Unit Testing

## Gitlab Runner
Gitlab kommt mit einem eigenen Continuous Integration System. Um Software aller Art zu testen und auszuliefern bietet es viele Möglichkeiten der Automatisierung.

### Konfiguration
```
# Select image from https://hub.docker.com/_/php/
image: php:7.1.1

stages:
  - build
  - test
  - deploy

# Caching
cache:
  paths:
  - vendor/

variables:
  # Configure mysql environment variables (https://hub.docker.com/r/_/mysql/)
  MYSQL_DATABASE: codeigniter_db
  MYSQL_ROOT_PASSWORD: mysql_strong_password

before_script:
  # Update packages and install composer and PHP dependencies.
  - apt-get update -yqq
  - apt-get install git libcurl4-gnutls-dev libicu-dev libmcrypt-dev libvpx-dev libjpeg-dev libpng-dev libxpm-dev zlib1g-dev libfreetype6-dev libxml2-dev libexpat1-dev libbz2-dev libgmp3-dev libldap2-dev unixodbc-dev libpq-dev libsqlite3-dev libaspell-dev libsnmp-dev libpcre3-dev libtidy-dev wget -yqq

  # Compile PHP, include these extensions.
  - docker-php-ext-install mbstring mcrypt pdo_mysql curl json intl gd xml zip bz2 opcache

  # Install XDebug
  - pecl install xdebug && docker-php-ext-enable xdebug 

  # Install Composer and project dependencies.
  - curl -sS https://getcomposer.org/installer | php
  - php composer.phar install

  # Install phpunit
  - wget https://phar.phpunit.de/phpunit.phar
  - chmod +x phpunit.phar
  - mv phpunit.phar /usr/local/bin/phpunit

  # Install ci-phpunit
  - php composer.phar require kenjis/ci-phpunit-test --dev
  - php vendor/kenjis/ci-phpunit-test/install.php

# Run our tests for php5.6
phpunit:php5.6:
  stage: test
  image: php:5.6
	services:
  	  - mysql:latest
  script:
    - wget https://phar.phpunit.de/phpunit-5.7.phar
    - mv phpunit-5.7.phar phpunit.phar
    - chmod +x phpunit.phar
    - mv phpunit.phar /usr/local/bin/phpunit

    - cd application/tests/
    - phpunit --coverage-text --colors=never

# Run our tests for php7.1
phpunit:php7.1:
  stage: test
  script:
    - cd application/tests/
    - phpunit --coverage-text --colors=never 
	services:
  	  - mysql:latest

# Building the projekt
building:
  stage: build
  script:
    - gulp

# Deploy App to the server
deploying:
  stage: deploy
  only:
    - master
```

Die gezeigte Konfiguration erzeugt zwei `jobs`. Einmal auf Basis von PHP5.6 und einmal auf PHP7.1 mit MySQL. 

### Basis-Image
Das Basis-Image ist `image: php:7.1.1`. Wenn `jobs` andere Images nutzen, müssen diese jeweils in den `jobs` definiert werden. Wenn kein Image im `job` angegeben wird, wird immer das Basis-Image genutzt.

### Vorbereitung
Der `before_script` dient als Vorbereitung. Hier sollten Abhängigkeiten installiert werden. 

```
before_script:
  - apt-get update -yqq
  - apt-get install git libcurl4-gnutls-dev libicu-dev
```

### Jobs
Der Gitlab Runner Dienst basiert auf einzelnen `jobs` welche durch Docker betrieben werden. Diese können auch lokal ausgeführt werden.
Es können unendlich viele `jobs` konfiguriert werden.

Jeder `job` kann frei konfiguriert werden. Somit ist es möglich, bei Web-Anwendungen, einzelne Umgebungen zu testen. Zum Beispiel die Application auf PHP5.6 und PHP7.1 oder ähnliches.

```
# Run our tests for php7.1
phpunit:php7.1:
  stage: test
  script:
    - cd application/tests/
    - phpunit --coverage-text --colors=never 
```

### Services
Jedem `job` kann einen oder mehrere `services` hinzugefügt werden, dabei können entweder Images von hub.docker.com genutzt werden:
`<docker-image>:<tag>` 

oder mittels `docker-compose` Befehle, auch eigene Umgebungen eingerichtet werden.
[Using Docker Images - GitLab Documentation](https://docs.gitlab.com/ce/ci/docker/using_docker_images.html#how-to-use-other-images-as-services) 

```
services:
  - postgres:9.3
  - mysql:5.6
```

### Stages
Es können mehrere Stages konfiguriert werden. Hierbei können die Stage-Namen frei gewählt werden. Die Reihenfolge der Stages ist entscheidend. Wenn eine Stage fehlschlägt, werden alle anderen danach abgebrochen. Im nachfolgenden Beispiel, würde ein Fehler in der `build`-Stage, die weiteren Stages `test` und `deploy` abbrechen. 

### Variables
Sie dienen dazu, für einzelne Services oder Aufrufe vordefinierte Werte bereitzustellen. So ist die Variable `MYSQL_DATABASE` zuständig für die Vergabe der Datenbank auf dem Service MySQL.

[MySQL - GitLab Dokumentation](https://docs.gitlab.com/ce/ci/services/mysql.html)
[MySQL Docker Hub Dokumentation](https://hub.docker.com/r/_/mysql/)

### Caching
Einzelne Ordner können zum Cache hinzugefügt werden um die Build-Zeit in Zukunft zu minimieren. 

### Fehler erlauben
Mit `allow_failure` kann einem `job` das fehlschlagen erlaubt werden. Dieser bricht dann nicht weitere jobs ab. 

### Branching
Mit `only` können jobs auf bestimmte Branches beschränkt werden. Zum Beispiel das deployen auf Produktiv-Server wird auf den master branch beschränkt.

### Weitere Dokumentation
[GitLab Documentation - Gitlab CI Yaml](https://docs.gitlab.com/ce/ci/yaml/README.html)


## CodeIgniter Testing
Als Basis für das testen dient das Framework PHPUnit. Um spezifisch auf CodeIgniter zu testen gibt es die Modifikation unter dem Link:
[GitHub - kenjis/ci-phpunit-test: An easier way to use PHPUnit with CodeIgniter 3.x.](https://github.com/kenjis/ci-phpunit-test)

Dieses wird durch die oben gezeigte CI-Konfiguration automatisch installiert. 

Es ist Standard, dass alle Test-Klassen über die Funktion 
`function setUp()` verfügen. Diese bereitet alle Daten für den Test vor.

Alle Möglichkeiten Werte und Variablen zu testen, findet sich unter: https://phpunit.de/manual/current/en/appendixes.assertions.html

### Controller Unit Test
Um Controller zu testen müssen die Unit-Tests im Ordner `application/tests/controller` abgelegt werden.

Die Controller können über `$this->request(‘GET’, ‘welcome/index’);` aufgerufen werden.

In dem unten aufgezeigten Beispiel, wird nach dem richtigen html Title-Tag gesucht. Um HTML UI zu testen, sollte jedoch auf andere Lösungen zurückgegriffen werden. Im Vordergrund beim testen des Controllers sollte die Datenverarbeitung stehen. Über POST Daten zum Controller zu senden und hinterher zu prüfen, ob die richtigen oder überhaupt Daten in die Datenbank geschrieben wurden.

Eine vollständige Dokumentation über alle Möglichkeiten beim Testen, findet sich hier: [ci-phpunit-test/HowToWriteTests.md at master · kenjis/ci-phpunit-test · GitHub](https://github.com/kenjis/ci-phpunit-test/blob/master/docs/HowToWriteTests.md)
[CI-PHPUnit Function Reference](https://github.com/kenjis/ci-phpunit-test/blob/master/docs/FunctionAndClassReference.md#testcaserequestmethod-argv-params--)

#### Beispiel
```php
class Welcome_test extends TestCase {
    
    public function setUp() {

    }

    public function test_index() {
        
        $output = $this->request('GET', 'welcome/index');
        
        $this->assertContains(
            '<title>Welcome to CodeIgniter</title>', $output
        );
    }
}
```

#### Beispiel 
```php
class Userinfos_test extends TestCase {
    
    public function setUp() {

    }

    public function test_post() {

        //Make Post request to controller
        $output = $this->request(
          'POST', 'form/index',
          ['name' => 'John Smith', 'email' => 'john@example.com']
        );

        $this->assertContains(
            '<title>Beispielseite - Registrierung erfolgreich</title>', $output
        );
    }
}
```


### Model Unit Test
Um Models zu testen, müssen die Unit-Tests im Ordner `application/tests/models` abgelegt werden.

Beim Model steht die Datenbank im Vordergrund. Hier sollte über Anfragen an das Model die Datenbank verändert werden.

Für bestimmte Testfälle, kann in der Konfiguration die DB initialisiert werden, z.b über `mysql -u root laravel < "sql/laravel.sql`.

#### Beispiel
```php
class Inventory_model_test extends TestCase {
    
    public function setUp() {
        
        $this->obj = $this->newModel('Inventory_model');
    }

    public function test_get_category_list() {
        
        $expected = [
            1 => 'Book',
            2 => 'CD',
            3 => 'DVD',
        ];
        
        $list = $this->obj->get_category_list();
        
        foreach ($list as $category) {
            $this->assertEquals($expected[$category->id], $category->name);
        }
    }

    public function test_get_category_name() {
        
        $actual = $this->obj->get_category_name(1);
        
        $expected = 'Book';
        $this->assertEquals($expected, $actual);
    }
}
```