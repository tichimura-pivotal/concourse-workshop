# Concourse Workshop for CTC

公式サイトはこちら
http://concourse.ci

## 事前準備 (Concourse環境がある前提としてます)
i. fly CLI (Fly Binaries)のインストール
- 講師から指定されるリンクにアクセスしてダウンロートしてください。
  注: Windowsは64bit環境を想定しておりますので、32bitの場合は別途おしらせください

### i. fly コマンドの利用
- ConcourseへのAdmin権限でのloginの実施

  - Concourse serverがdeployされている場合

  ``
  fly -t demo login -c http://指定されたURL:8080
  ``
  (8080 portで受け付けていることを想定)
　targetは各自異なりますのでご注意ください
　
- 参考: Concourseへのteamの登録(BASIC認証)

  注: Adminでのログインが必要です。(Basic認証の場合は複数ユーザ登録ができません)

  ``
  $ fly -t demo set-team -n team1 --basic-auth-username=user --basic-auth-password=pass
  Team Name: team1
  Basic Auth: enabled
  GitHub Auth: disabled
  UAA Auth: disabled
  Generic OAuth: disabled
  apply configuration? [yN]: y
  team created  
  ``

- Concourseへのユーザloginの実施

  ``
  fly -t demo login -c <target> -n <team> -u <user> -p <pass>
  ``
  　　
#### pipelineの作成

- YAMLファイルの作成(1)

  以下を参照にファイルを作成

  ```
  jobs:
  - name: hello-world
    plan:
    - task: say-hello
      config:
        platform: linux
        image_resource:
          type: docker-image
          source: {repository: ubuntu}
        run:
          path: echo
          args: ["Hello, world!"]
  ```

 上記ファイルをhello.ymlとして作成、パイプラインを実行
 （login時にdemoというtargetが設定されている前提)

 ``
 fly -t demo set-pipeline -p hello-world -c hello.yml
 ``

 パイプラインがpauseした状態なので、下記のコマンドを打つか、
 ボタン(再生ボタンのような)を押す

 ``
fly -t demo unpause-pipeline -p hello-world
  ``

  "hello world" jobをクリックして、右側のプラスボタンを押下

- YAMLファイルの作成(2)

  以下を参照にファイルを作成

  ```
  resources:
  - name: every-1m
    type: time
    source: {interval: 1m}

  jobs:
  - name: navi
    plan:
    - get: every-1m
      trigger: true
    - task: annoy
      config:
        platform: linux
        image_resource:
          type: docker-image
          source: {repository: ubuntu}
        run:
          path: echo
          args: ["Hey! Listen!"]
  ```

  これをnavi-pipeline.ymlとして保存し、以下のコマンドを実行します。

  >  先に作成したパイプラインを上書きします。

　``
  fly -t demo set-pipeline -p hello-world -c navi-pipeline.yml
  ``

  同じようにpausedになっているので、unpause-pipelineを実行するか、アイコンをクリックして下さい。

#### JobとPlan

  - 各Jobは必ず一つのPlanを持つ
  - Planの中で、Resourceからの読込や更新、あるいはTaskの実行を行う

#### Resource

 - Resourceはあらかじめ定義、あるいは、個別に定義が可能
 - 個別に定義する場合は、resource_typesにて定義

#### Task

 - inputsとoutputsを定義して、タスクとして実施する内容を記述
 - 必須項目としては、
    - platform
    - run
 - fly CLIのexecuteコマンドから実行も可能

#### Cloud Foundryへの展開

 cfにアプリケーションをプッシュします
 - cf-simple.yml

```
resources:
- name: cf-sample-app-spring
  type: git
  source:
    uri: https://github.com/cloudfoundry-samples/cf-sample-app-spring.git

- name: cf-target
  type: cf
  source:
    api: {{cf_api}}
    username: {{cf_user}}
    password: {{cf_user_pass}}
    organization: {{cf_org}}
    space: {{cf_space}}
    skip_cert_check: true

jobs:
- name: deploy CF
  plan:
  - get: cf-sample-app-spring
    trigger: true
  - put: cf-target
    params:
      manifest: cf-sample-app-spring/manifest.yml
      path: cf-sample-app-spring
```

  - cf-simple-settings.yml

```
cf_api: https://api.run.pivotal.io
cf_user: $CI_CFUSER
cf_user_pass: $CI_CFPASS
cf_org: $CI_ORG
cf_space: $CI_SPACE
```

  - fly コマンドの実行

``
fly -t demo set-pipeline -p hello-cf -c cf-simple.yml -l cf-simple-settings.yml
``

#### Tocaroとの連携

 cfにアプリケーションをプッシュしたパイプラインを変更して、tocaroにプッシュします.
 tocaroのchannel_idに対して通知が行われますので、下記の``"メッセージを入力"``の部分を適宜変更してください.
 なお、最後の``:``は必要ですので残してください.

 ```
 on_success:
   put: tocaro-alert
   params:
     text: |
       "メッセージを入力":
 ```


 - cf-sample-tocaro.yml

```
resources:
- name: cf-sample-app-spring
  type: git
  source:
    uri: https://github.com/tichimura-pivotal/cf-sample-app-spring.git
  check_every: 10s # demo only

- name: cf-target
  type: cf
  source:
    api: {{cf_api}}
    username: {{cf_user}}
    password: {{cf_user_pass}}
    organization: {{cf_org}}
    space: {{cf_space}}
    skip_cert_check: true

- name: tocaro-alert
  type: tocaro-notification
  source:
    url: {{TOCARO_CHANNEL_ID}}

resource_types:
- name: tocaro-notification
  type: docker-image
  source:
    repository: tichimura/tocaro-notify-resource
    tag: latest

jobs:
- name: deploy CF
  plan:
  - get: cf-sample-app-spring
    trigger: true
  - put: cf-target
    params:
      manifest: cf-sample-app-spring/manifest.yml
      path: cf-sample-app-spring
  on_success:
    put: tocaro-alert
    params:
      text: |
        "メッセージを入力":
  on_failure:
    put: tocaro-alert
    params:
      text: |
        The deploy has failed...:
```

 設定ファイルはこちらです。TOCARO_CHANNEL_IDは以下より取得できますので、$TOCARO_CHANNEL_IDの値を変更してください。

 https://tocaro.im/user/integrations

 ``Webhook``　> 設定する
 投稿先グループ
 Webhook URL
 ``クリップボードにコピー``を選択

  - cf-sample-tocaro-settings.yml

```
cf_api: https://api.run.pivotal.io
cf_user: $CI_CFUSER
cf_user_pass: $CI_CFPASS
cf_org: $CI_ORG
cf_space: $CI_SPACE
TOCARO_CHANNEL_ID: $TOCARO_CHANNEL_ID
```

  - fly コマンドの実行

``
fly -t demo set-pipeline -p hello-tocaro -c cf-simple.yml -l cf-simple-settings.yml
fly -t demo unset-pipeline -p hello-tocaro
``

  - (Option) git pushからの通知
  git pushによるコード変更を受けて、通知が来ることを確認します。


  - fly -t demo destroy-pipeline -p hello-tocaro
  不要なpushを避けるため、pipelineをとめます。

  - cf delete <app-name>
  cf applicationを削除します。


#### Java プロジェクトとしての展開

  以下を参考にして、パイプラインを構成してみましょう。

[https://github.com/Pivotal-Field-Engineering/PCF-demo/tree/master/ci]

  1. プロジェクトのfork

  https://github.com/Pivotal-Field-Engineering/PCF-demo.git

  2. git cloneの実施

``
git clone https://github.com/<your-git-name>/PCF-demo.git
``

　3. Unitタスクの実行

```
$  cd PCF-demo
$ fly -t demo execute -c ci/tasks/unit.yml -i pcfdemo=.
```

  > 通常では稼働しているディレクトリをINPUT名として利用が可能
  > taskにてインプット名を別でしているため、このような指定が必要
  > (フォルダ名はPCF-demoだが、インプット名はpcfdemo)
  > そのため、-i pcfdemo=. として明記


  　3. Buildタスクの実行

  ```
  $ mkdir -p target/version
  $ echo "1.0.0-rc.1" > target/version/number
  $ fly -t demo execute -c ci/tasks/build.yml -i pcfdemo=. -i version=target/version
  ```


  　4. ciディレクトリのファイルを編集

  ```
  $ mkdir ~/.concourse
  $ cp ci/pcfdemo-properties-sample.yml ~/.concourse/pcfdemo-properties.yml
  $ chmod 600 ~/.concourse/pcfdemo-properties.yml
  ```

  　5. 設定ファイルの編集

  ``
  ~/.concourse/pcfdemo-properties.yml
  ``


  　6. pipelineファイル(pipeline.yml)の確認

  ``
  ci/pipeline.yml
  ``


  　7. pipelineの設定

  ``
  fly -t demo set-pipeline -p pcfdemo -c ci/pipeline.yml -l ~/.concourse/pcfdemo-properties.yml
  ``


  　8. unpause-pipelineの実行

  ``
  fly -t demo unpause-pipeline -p pcfdemo
  ``


  　9. ship-itタスクの実行

  ``
  $ fly -t demo trigger-job -j pcfdemo/ship-it
  ``

#### プロジェクト管理ツールとの連携
- pipelineの構成(YAMLの作成)

```
resources:
- name: git-repo-path
  type: git
  source:
    uri: YOUR_GITHUB_REPOSITORY

- name: tracker-output
  type: tracker-resource
  source:
    token: TRACKER_API_TOKEN
    project_id: "TRACKER_PROJECT_ID"
    tracker_url: https://www.pivotaltracker.com

resource_types:
- name: tracker-resource
  type: docker-image
  source:
    repository: concourse/tracker-resource

jobs:
- name: deploy
  plan:
  - get: git-repo-path
  - put: tracker-output
    params:
      repos:
      - git-repo-path
```

- pipelineの登録
(TARGETは、ご自身の環境に合わせてご用意下さい)

  ``
  fly -t TARGET set-pipeline -p trackerdemo -c gittracker.yml -l gittracker.settings.yml
  ``

- Storyの作成と、commitの実施

  - github上でpivotal trackerとの連携( "Settings" -> "Integration & Services" )
  - story idの作成と開始(Startedになっていることを確認)
  - story idのコピー
  - git commit -m "XXXXXXXX [(Finishes|Finished) #ID]"
  - story の状態が指定した通りに変わっている事を確認(Started->Finished)
  - pipelineに登録した内容により　Deliveredになる事を確認

#### 自習

- https://github.com/Pivotal-Field-Engineering/PCF-demo/tree/master/ci
- http://concourse.ci/flight-school.html
- https://github.com/concourse/atc/blob/master/routes.go#L126

#### Taskの実行結果とアクション

  Taskの実行結果によって、取りうるアクションを決められます。

  - on_successの利用

```
  plan:
  - get: foo
  - task: unit
    file: foo/unit.yml
    on_success:
      task: alert
      file: foo/alert.yml  
  ```

  上記は、下記と同じ内容ですが、後述のon_failureとの使い分けを考えると上記のon_successを使うのが推奨されます。

  ```
  plan:
  - get: foo
  - task: unit
    file: foo/unit.yml
  - task: alert
    file: foo/alert.yml  
  ```

  - on_failureの利用

```
  plan:
  - get: foo
  - task: unit
    file: foo/unit.yml
    on_failure:
      task: alert
      file: foo/alert.yml
  ```

  ```
  plan:
  - get: foo
  - task: unit
    file: atomy-pr/ci/unit.yml
    on_success:
      put: atomy-pr
      params:
        path: atomy-pr
        status: success
    on_failure:
      put: atomy-pr
      params:
        path: atomy-pr
        status: failure  
  ```

#### 通知ツールとの連携

```
resource_types:
- name: slack-notification
  type: docker-image
  source:
    repository: cfcommunity/slack-notification-resource
    tag: latest
...
- name: slack-alert
  type: slack-notification
  source:
    url: https://hooks.slack.com/services/xxxx
...
on_success:
  put: slack-alert
  params:
    text: |
      The deploy has done...:
on_failure:
  put: slack-alert
  params:
    text: |
      The deploy has failed...:
```

#### Jobの管理

  ```
  # -j PIPELINE/JOB
  fly -t demo watch -j hello-cf/"deploy CF"
  ```


#### 参考 Authの取り方

type(Bearer)からの入力が必要

token="$(curl -u concourse:changeme http://192.168.99.100:8080/api/v1/teams/main/auth/token | jq -r '.type + " " + .value')"
