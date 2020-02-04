## APIモードのRailsをDocker上で動作させる

https://docs.docker.com/compose/rails/

`$ touch Dockerfile docker-compose.yml Gemfile Gemfile.lock entrypoint.sh`

``` Dockerfile
FROM ruby:2.6
RUN apt-get update -qq && apt-get install -y nodejs postgresql-client
RUN mkdir /app
WORKDIR /app
COPY Gemfile /app/Gemfile
COPY Gemfile.lock /app/Gemfile.lock
RUN bundle install
COPY . /app

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Start the main process.
CMD ["rails", "server", "-b", "0.0.0.0"]
```

``` docker-compose.yml
version: '3'
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/app
    ports:
      - "3000:3000"
    depends_on:
      - db
```

あとで上書きされる。Gemfile.lockは空でOK
``` Gemfile
source 'https://rubygems.org'
gem 'rails', '~>5'
```

``` entrypoint.sh
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /app/tmp/pids/server.pid

# Then exec the container’s main process (what’s set as CMD in the Dockerfile).
exec "$@"
```

用意できたら
`$ docker-compose run web rails new . -—api -—force -—no-deps —-database=postgresql`

`Overwrite /app/Gemfile? (enter "h" for help) [Ynaqdhm]`
が出たら Y でOKです。

`$ docker-compose build`

完了したらDB接続設定をする
`config/database.yml`

``` database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password:
  pool: 5

development:
  <<: *default
  database: app_development

test:
  <<: *default
  database: app_test
```

書いたら

```
$ docker-compose run web rake db:create
$ docker-compose up
```

DBが作成されたら完了。

`http://localhost:3000`でRailsが動作しているはず。
