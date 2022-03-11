# Amazon Linux AMI であることの確認
```
cat /etc/os-release
```

# Docker環境削除
```
docker rmi `docker images -q`
```

# PHP 7.2系へアップグレード
```
php -v
sudo yum -y install http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
sudo yum -y install php72 php72-cli php72-common php72-devel php72-gd php72-intl php72-mbstring php72-mysqlnd php72-pdo php72-pecl-mcrypt php72-opcache php72-pecl-apcu php72-pecl-imagick php72-pecl-memcached php72-php-pecl-redis php72-php-pecl-xdebug php72-xml
sudo alternatives --set php /usr/bin/php-7.2
php -v
```

# PDOがインストールされたことの確認
```
php -m | grep pdo

```

# メモリ確保のためのswap領域作成
```
sudo sh -c "echo 3 > /proc/sys/vm/drop_caches"
sudo dd if=/dev/zero of=/var/swap.1 bs=1M count=1024
sudo chmod 600 /var/swap.1
sudo mkswap /var/swap.1
sudo swapon /var/swap.1
sudo cp -p /etc/fstab /etc/fstab.ORG
sudo sh -c "echo '/var/swap.1 swap swap defaults 0 0' >> /etc/fstab"
```

# システム時間をUTCから日本時間に変更
```
echo "Asia/Tokyo" | sudo tee /etc/timezone
sudo mysql_tzinfo_to_sql /usr/share/zoneinfo
sudo cp /etc/sysconfig/clock /etc/sysconfig/clock.org

sudo vi /etc/sysconfig/clock

sudo ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
sudo reboot
```

# MySQLの文字化け禁止設定
```
sed -e "/utf8/d" -e "/client/d" -e "/^\[mysqld_safe\]$/i character-set-server=utf8\n\n[client]\ndefault-character-set=utf8" /etc/my.cnf |sudo tee /etc/my.cnf
```

# MySQLの起動とrootでのログイン
```
sudo service mysqld start
mysql -u root

mysql> show variables like '%char%';
```

# MySQLの中にデータベースとテーブル作成
```
mysql> show databases;
mysql> create database user_register character set utf8;
show databases;

mysql> use user_register;
mysql> show tables;

create table users(
    id int primary key auto_increment, 
    name varchar(50) not null,
    age int not null,
    gender varchar(10) not null,
    created_at timestamp default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

mysql> insert into users(name, age, gender) values('iwai', 23, 'male');
mysql> insert into users(name, age, gender) values('harada', 18, 'female');
mysql> select * from users;
```

# Modelクラスの作成
```
<?php
    // モデルのスーパークラス
    class Model {
    
        // データベースと接続を行うメソッド
        protected static function get_connection(){
            try {
                // オプション設定
                $options = array(
                    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,        // 失敗したら例外を投げる
                    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_CLASS,   //デフォルトのフェッチモードはクラス
                    PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES utf8',   //MySQL サーバーへの接続時に実行するコマンド
                );
                
                $pdo = new PDO('mysql:host=localhost;dbname=user_register', 'root', '', $options);
                return $pdo;
                
            } catch (PDOException $e) {
                return 'PDO exception: ' . $e->getMessage();
            }
        }
        
        // データベースとの切断を行うメソッド
        protected static function close_connection($pdo, $stmt){
            try {
                $pdo = null;
                $stmt = null;
            } catch (PDOException $e) {
                return 'PDO exception: ' . $e->getMessage();
            }
        }
    }
```

# User モデルの拡張
```
<?php
    require_once 'models/Model.php';
    // モデル(M)
    // ユーザーの設計図
    class User extends Model{
        // プロパティ
        public $id; // ID
        public $name; // 名前
        public $age; // 年齢
        public $gender; // 性別
        public $created_at; // 登録日時
        // コンストラクタ
        public function __construct($name='', $age='', $gender=''){
            // プロパティの初期化
            $this->name = $name;
            $this->age = $age;
            $this->gender = $gender;
            // print $this->name . 'が生まれた' . PHP_EOL;
            // print "{$this->name}が生まれた\n";
        }
        public function drink(){
            if($this->age >= 20){
                return 'お酒OK' . PHP_EOL;
            }else{
                return 'お酒NG' . PHP_EOL;
            }
        }
        
        public function validate(){
            $errors = array();
            if($this->name === ''){
                $errors[] = '名前を入力してください';
            }
            if($this->age === ''){
                $errors[] = '年齢を入力してください';
            }else if(!preg_match('/^[0-9]+$/', $this->age)){
                $errors[] = '年齢は正の整数で入力してください';
            }
            
            return $errors;
        }
        
        // 全テーブル情報を取得するメソッド
        public static function all(){
            try {
                $pdo = self::get_connection();
                $stmt = $pdo->query('SELECT * FROM users ORDER BY id DESC');
                // フェッチの結果を、Userクラスのインスタンスにマッピングする
                $stmt->setFetchMode(PDO::FETCH_CLASS|PDO::FETCH_PROPS_LATE, 'User');
                $users = $stmt->fetchAll();
                self::close_connection($pdo, $stmt);
                // Userクラスのインスタンスの配列を返す
                return $users;
            } catch (PDOException $e) {
                return 'PDO exception: ' . $e->getMessage();
            }
        }
        
        // データを1件登録するメソッド
        public function save(){
            try {
                $pdo = self::get_connection();
                $stmt = $pdo->prepare("INSERT INTO users (name, age, gender) VALUES (:name, :age, :gender)");
                // バインド処理
                $stmt->bindParam(':name', $this->name, PDO::PARAM_STR);
                $stmt->bindParam(':age', $this->age, PDO::PARAM_INT);
                $stmt->bindParam(':gender', $this->gender, PDO::PARAM_STR);
                // 実行
                $stmt->execute();
                self::close_connection($pdo, $stmt);
                return "新規ユーザー登録が成功しました。";
                
            } catch (PDOException $e) {
                return 'PDO exception: ' . $e->getMessage();
            }
        }
    }
```
