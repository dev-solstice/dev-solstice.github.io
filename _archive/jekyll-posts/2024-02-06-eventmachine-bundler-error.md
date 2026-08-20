---
layout: single
title: "An error occurred while installing eventmachine (1.2.7), and Bundler cannot continue."
typora-root-url: ../

---

# An error occurred while installing eventmachine (1.2.7), and Bundler cannot continue.



EventMachine 이 필요로 하는 종속성 설치

```
brew install openssl libyaml
```



EvenMachine 재설치

```
gem install eventmachine -v '1.2.7' -- --with-cppflags=-I/usr/local/opt/openssl/include --with-ldflags=-L/usr/local/opt/openssl/lib
```



Openssl 업데이트

```
bundle config build.eventmachine --with-ssl-dir=/opt/homebrew/opt/openssl@3 
```



참고



[Gem::FilePermissionError 에러](https://jojoldu.tistory.com/288)

[bundle install 작동불가](https://stackoverflow.com/questions/75944204/macos-ventura13-3-with-bundle-install-eventmachine-stuck-in-failing-loop)

