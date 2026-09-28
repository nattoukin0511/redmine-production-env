#概要
Redmineの構築を通して、本番環境の構築手順やトラブルシューティングのプロセスを成果物としてまとめる。

---
#1.構築手順
 # Ruby
 rbenv install 3.2.2
 rbenv global 3.2.2

 # Gem
 gem install bundler
 gem install passenger

 # Gem導入
 bundle install --without development test

 # DB作成
 RAILS_ENV=production bundle exec rake db:migrate

 # 初期データ
 RAILS_ENV=production bundle exec rake    redmine:load_default_data

 # Apache反映
 sudo a2enmod passenger
 sudo a2ensite redmine.conf

 # 再起動
 sudo systemctl restart apache2

---
#2発生したエラー
1. Apache起動エラーSomething went wrong error
　症状
　　　Apache再起動失敗
   
2. Passenger動作不良
　症状
　　　Redmine画面が表示されない
   
3. Ruby/Bundler不整合
  症状
　　　bundle install失敗
　　　PassengerRuby参照先不一致
   
4. DB未作成 DB doesn't exit Table
5. 初期データ未投入
6. secret_key_base問題

---
#3原因調査
 1　journalctl -xeu apache2
　　apache2ctl configtest

 2　apache2ctl -M | grep passenger
　  passenger-status

 3 which ruby
   ruby -v
　 passenger-config --ruby

 4　RAILS_ENV=production bundle exec rake db:migrate

 5 RAILS_ENV=production bundle exec rake redmine:load_default_data

 6 rails credentials:edit
generate_secret_token

---
#4解決方法
 #1.Apache起動エラーとPassenger動作不良
/etc/apache2/mods-available/passenger.conf
PassengerRuby /usr/bin/ruby
/homeから/usr/rubyへ

 sudo chown -R www-data:www-data /var/www/redmine
 sudo chmod -R 755 /var/www/redmine
 cd /var/www/redmine

 3．rbenv切替
  rm -rf ~/.rbenv/versions/3.3.0

 4.RAILS_ENV=production を付けて、本番環境向け設定でDBマイグレーションと初期データ投入を実施しました。
 RAILS_ENV=production bundle exec rake db:migrate

 5.RAILS_ENV=production bundle exec rake    redmine:load_default_data
 
 6.sudo nano secret_token.rb
 ruby -e "require 'securerandom'; puts SecureRandom.hex(64)
---
#5.修正ファイル
　/etc/apache2/mods-available/passenger.conf
　/etc/apache2/sites-available/redmine.conf
/etc/systemd/system/apache2.service.d/override.conf
/var/www/redmine/config/initializers/secret_token.rb
config/credentials.yml.enc
---
#6.学んだこと
 権限・所有者の見直し: www-data への所有者変更と パーミッションの適切な設定により、Passengerからのアクセス権エラーを解消。

 PassengerRuby設定する際に気を付けること
 本番環境やステージング環境のWebサーバー（Apache/Nginx）を設定する際、`PassengerRuby`（または `passenger_ruby`）のパスに `~/.rbenv/shims/ruby` や `/home/ユーザー名/...` 以下のパスを指定しないでください。

* **理由:** Webサーバー（`www-data` や `nginx` ユーザー）には、各ユーザーの `/home` ディレクトリ内を実行する権限（パーミッション）がないため、サイトが起動せずエラーになります。また、環境変数が引き継がれないため `shims` は正常に動作しません。

* **対策:** 必ず `rbenv` でインストールされた**本物のRubyバイナリのフルパス**を指定してください。例 /usr/bin/ruby


 Rubyバージョンの整合性: 意図しないバージョン（3.3.0等）が混ざっていたため、環境を整理して対応。

 本番環境の明示: RAILS_ENV=production を付与して db:migrate や redmine:load_default_data を実行し、本番DBを正しく構築。

 シークレットキーの生成: SecureRandom.hex(64) を利用して安全なトークンを作成・設定。
