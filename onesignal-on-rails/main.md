この記事は信州大学kstmアドベントカレンダーの **20** 日目で、私の誕生日である12月の **20** 日目に公開され、 **21** 歳を迎えることとなりました:tada:
~~20と20と21という惜しい感じが言いたかっただけ。~~

# はじめに

最近はWeb関連の技術が人気であり、Webアプリ・サービスも山のようにあります。
また、大抵のWebサービスにはネイティブアプリ（すなわちiOSやAndroidのアプリなど）があることも、今日では当然のように思っていますよね。
でも疑問が浮かびませんか？

- なぜ同じ内容のものを2つも用意するのでしょうか？
- ブラウザで利用できるだけではダメなのでしょうか？
- 何か優位なものがあるのでしょうか？

ここで、Webサービスとネイティブアプリとの間には **通知手法の相違** ということがあると私は考えました。
Webサービスにおいては登録されたメールアドレスに対する **メール通知** が主な通知手段であり、ネイティブアプリでは端末に対する **プッシュ通知** が利用されています。
メールはフォーマルな連絡手段なので、通知として利用されるのは~~うざい~~よくないですよね。
また、通知をさせるためだけにネイティブアプリを作るのは面倒ですよね。

そこで、Webアプリに手取り早く通知機能を持たせられる[OneSignal](https://onesignal.com/)を用いたWebプッシュ通知を紹介します。

# 環境

- macOS 10.13.6
- ruby 2.5.1
- Rails 5.2.1

# 雛形作り

簡素なRailsアプリを作ります。
今回はScaffoldでPostモデルを作り、CURDがされるたびに通知がされるようなものを作って行きます。

```shell-session
$ rails new -B -T -M -C onesignal-on-rails
$ cd onesignal-on-rails
```

OneSignalのapiを叩くことになるので、HTTPartyをGemfileに追加して `bundle install` します。

```diff
# Gemfile

+ gem 'httparty'
```

```shell-session
$ bundle install --path=vendor/bundle
```

余談なんですが、railsのプロジェクト作る時ってグローバルインストールする人が多いんですかね？
nodeとかと同じようにデフォルトで `node_modules` みたくローカルにインストールする仕様の方がいいと思うんだけどなあ...

Scaffoldで雛形を作ります。

```shell-session
$ bundle exec rails g scaffold post title:string content:text
$ bundle exec rails db:migrate
```

```ruby:config/routes.rb
Rails.application.routes.draw do
  resources :posts
  root to: 'posts#index' # 追加
end
```

`bundle exec rails s` 後に `localhost:3000` にて表示が確認できれば完了です。

# OneSignalの設定

まず[OneSignal](https://onesignal.com/)へSignupしましょう。
GitHubアカウントででもいけます。

サインアップ後、ADD APPします。
画像

するとプラットフォームを問われるので今回はWeb Pushを選択します。
画像

次に各種項目を設定します。

- Choose Integration は Typical Site
- Site URL は `localhost:3000`
- Permission Prompt には Subscription Bell, HTTP-Pop-Up

Site nameやChoose a labelは適宜指定してください。
Permission Promptの設定から表示させる文言の編集なども可能です。
画像

設定が終わったらSaveをし、表示されるコードをコピーします。
コピーしたコードは `application.html.erb` のheadタグ内部へ若干の変更を加えてペーストします。

```html:app/views/layouts/application.html.erb
<!DOCTYPE html>
<html>
  <head>
    <title>OnesignalOnRails</title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <!-- ここから -->
    <script src="https://cdn.onesignal.com/sdks/OneSignalSDK.js" async=""></script>
    <script>
      var OneSignal = window.OneSignal || [];
      OneSignal.push(function() {
        OneSignal.init({
          appId: "your-app-id",
          notifyButton: {
            enable: true,
          }
        });
      });
    </script>
    <!-- ここまで -->

    <%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track': 'reload' %>
    <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
  </head>

  <body>
    <%= yield %>
  </body>
</html>
```

# 通知の生成

CURDそれぞれの動作が行われた時に通知をさせるので、ServiceObjectとしてこの処理をまとめておきます。
`your-rest-api-key` はOneSignalのAccountManagementから確認できます。

```ruby:app/service/create_notification.rb
class CreateNotification
  include HTTParty

  API_URI = 'https://onesignal.com/api/v1/notifications'.freeze

  def self.call(*args)
    new(*args).call
  end

  def initialize(contents:, type:)
    @contents = contents
    @type     = type
  end

  def call
    HTTParty.post(API_URI, headers: headers, body: body)
  end

  private

  attr_reader :contents, :type

  def headers
    {
      'Authorization' => 'Basic your-rest-api-key',
      'Content-Type'  => 'application/json'
    }
  end

  def body
    {
      'app_id' => 'your-app-id',
      'url'    => 'localhost:3000',
      'data'   => { 'type': type },
      'included_segments' => ['All'],
      'contents' => contents
    }.to_json
  end
end
```

あとはControllerの任意のアクションにて `CreateNotification.call` を行うのみです。

```ruby:app/controllers/posts_controller.rb
def create
  ...

  CreateNotification.call(
    contents: { 'en' => 'Post created!', 'ja' => 'ポストが作成されました！' },
    type: 'posts#create'
  )
  ...
end

def update
  ...
  CreateNotification.call(
    contents: { 'en' => 'Post updated!', 'ja' => 'ポストが更新されました！' },
    type: 'posts#update'
  )
  ...
end

def destroy
  ...
  CreateNotification.call(
    contents: { 'en' => 'Post destroyed!', 'ja' => 'ポストが削除されました！' },
    type: 'posts#update'
  )
  ...
end
```

# 確認

`bundle exec rails s` でサーバーを起動後 `localhost:3000` へアクセスします。
しばらくすると右下にベルマークが出るので、そこから通知を許可します。
画像

許可設定がうまくいくと、（設定していれば）Welcome messageが送信されてきます。
画像

新しくポストを作れば...
画像

通知がきますね:tada:

ユーザーが複数いたら（異なるブラウザ）...
画像

同じように通知が届いています:tada::tada:

# おわりに

Webプッシュ通知を簡単に導入できるOneSignalについて紹介しました。
APIを叩くことで通知を送れるほか、通知を送る相手の指定やWeb上からも通知の送信ができます。
画像

今回はWebプッシュ通知でしたが、プラットフォームとしてネイティブアプリやメールにも対応しているそうです。
ぜひ試してみてください:tada:

# 参考

[Sending Web Push Notifications with OneSignal and Ruby on Rails · Sweetcode.io](https://sweetcode.io/web-push-notifications-onesignal-ruby-rails/)
