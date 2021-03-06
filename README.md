![comics image](http://i.imgur.com/yIJr3tX.png)

# Ускорение тестов.
Как известо ruby сообщество является одним из наиболее тестолюбивых. И при разработке средних и больших проектов количество тест кейсов может легко перевалить за несколько тысяч или даже десятков тысяч. С ростом количества происходит и увеличение времени выполнения всех тестов. И если у вас меньше ~300-500 тест кейсов, вы могли запустить на выполнение их всех и через минуту получить результат. Но с количеством больше 1000 вам придется ждать значительно дольше. И если при разработке вы используете принцип TDD и Continuous Integration, такие задержки будут довольно критичными. Я попытаюсь рассказать, каким способами можно сократить это время.

#### Применение гемов, специально разработанных для ускорения запуска ruby-процессов, в том числе и тестов.
Если требуется обеспечить Continuous Integration на вашей локальной машине, вы можете несколько ускорить время выполнения тестов за счет использования гемов [zeus](https://github.com/burke/zeus) или [spring](https://github.com/rails/spring) (по дефолту начиная с Rails 4.1). Они загружают окружения в память и при запуске теста он начинает выполнятья мгновенно. Это может сэкономить вам до 15секунд.

Так же вы можете распаралеллить выполнение ваших тестов с помощью [parallel_tests](https://github.com/grosser/parallel_tests). Y меня с моим 2-ядерным процессором получилось выиграть всего 40% времени.

Еще вы можете использовать гем [guard](https://github.com/guard/guard), который будет отслеживать изменения выших файлов и выполнять только те тесты, на которые эти изменения могут повлиять. Для этого вам нужно настроить наблицу матчинга "файлы - тесты".

Использование этих инструментов подробно описано в скринкасте [Ryan Bates] (http://railscasts.com/episodes/412-fast-rails-commands)`a

#### Настройка конфигурации тестового окружения.

Для начала вы можете настроить `spec_helper.rb` для запуска тесов, исключив медленные. Для этого вам нужно выяснить, какие из ваших тестов медленные. Для этого запутстите `bundle exec rspec -p`. Эта команда выполнить все спеки и выведет время, затраченное на выполнение каждого файла спеков.

Вот такую [информацию]( https://gist.github.com/anykey007/e3251b9c317944abbbb4) вывело у меня.

Как видите, самыми медленными оказались интеграционые тесты (за счет длительного времени на выполнение запроса, рендеринг страницы) и тесты для статистики (за счет генерации большого количества данных для БД). Потому я помечу их как `:slow`

Для этого добавлю в `spec_helper.rb` эту строку 
```ruby
config.filter_run_excluding :slow unless ENV["SLOW_SPECS"]
```
а в файле тестов добавлю метку `:slow` возле медленных examples. например 
```ruby
context 'without start date', :slow do
```

Так же вы можете пометить весь файл как `:slow`
```ruby
describe Reports::CoachUsageStatistics, :slow do
```
Также вы можете отключать сборщик мусора. Для тестов он не так важен т.к. предназначен для освобождения памяти от объектов, которые уже не востребованы. Безусловно, когда процесс постоянно висит в памяти, без GC у нас бы очень быстро забилась оперативная память. Но в нашем случае тесты выполнются, после чего процесс, их запустивший, удаляется. А после его удаления освобождается и память, занятая им. Этим уже занимается ОС.
```ruby
config.before(:each) { GC.disable }
config.after(:each)  { GC.enable }
```
```
Finished in 2 minutes 47.5 seconds
Finished in 2 minutes 39.4 seconds
```

#### Для быстрых тестов необходимо правильно их писать. Это будет наш третий способ.

Возьмем за пример файл https://gist.github.com/anykey007/cbd170fdf8075a2c0a84
Он выполняется довольно долго, потому что перед каждым тест кейсом выполняется блок 
```ruby
bebore(:each)
```
```
Finished in 2 minutes 13.7 seconds
20 examples, 0 failures
```
Но если мы объединим некоторые тесты как здесь https://gist.github.com/anykey007/8846489a194e7271ecfd

Поучим неплохое увеличение скорости
```
Finished in 40.19 seconds
6 examples, 0 failures
```
Да, это не совсем [best practices](http://betterspecs.org/), так как рекомендуют создавать отдельный example для каждого тестируемого значения. Но в нашем случае для проведения теста создается большое количество тестовых данных, что значительно увеличивает время его прохождения. Потому я считаю, что в таких случаях можно пойти на копромис. В итоге я сэкономил почти 1.5 минуты на одном файле.

В тестах статистики мы можем миновать создание записей с помощью `activerecord` и использовать `native sql`, что должно еще больше ускорость выполнения теста.
Обновив до https://gist.github.com/anykey007/9f6227d6c7feb7c6f8cb имеем результат
```
Finished in 10.76 seconds
4 examples, 0 failures
```
Имеем ускорение больше чем в 10 раз.

#### Правильное создание ActiveRecord объектов.

Если вы используете [factory_girl](https://github.com/thoughtbot/factory_girl), то должны знать, что в отличии от фикстур при вызове метода `create` будет создана реальная запись в БД. И это довольно накладно, особенно если в фабриках прописаны дополнительно зависимости. Потому создатели [factory_girl](https://github.com/thoughtbot/factory_girl) рекомендуют использовать вместо `create` методы `build` и `build_stubbed` в тех местах, где не требуется сохранение объекта. Например, проверка валидации или связей.

Для ускорения работы [factory_girl](https://github.com/thoughtbot/factory_girl) можно использовать [factory_girl-seeds](https://github.com/evrone/factory_girl-seeds), где вместо `create` мы используем метод `seed`, который не создает новую запись, а просто загружает. Предварительно мы должны создать эту запись, указав в настройках. Она будет загружена до начала выполнения тестов. По словам создателей это может уменьшить общее время выполнения тестов на 50%.

Я поместил в `spec_helper.rb` загрузгу двух фабрик, создающих пользователей которые я использую в одном из spec файлов.
```ruby
config.before(:suite) do
  FactoryGirl::SeedGenerator.create(:user)
  FactoryGirl::SeedGenerator.create(:user_with_session)
end
```

Запустив тест я получил следующий результат.
```
Finished in 14.35 seconds
74 examples, 0 failures
```
До этого он выполнялся медленней.
```
Finished in 23.11 seconds
74 examples, 0 failures
```
Все эти приемы, конечто же, не дадут той большой скорости, как при тестах с использованием фикстур. Но фикстуры не очень элегантны в сравнении с фабриками, в которых мы может добавить faker для генерации полей, определить ассоциации и колбеки `after(:create)`, `after(:build)`, etc. Потому я нахожу идею создания фикстур на основе фабрик довольно интересной.

Существует гем [fixture_builder](https://github.com/rdy/fixture_builder). С его помощью мы можем конвертировать фабрики в фикстуры, указав все это в конфигурации.

Но это не совсем юзабельно. Необходимо описать фабрику, вызвать метод её создания в файле конфигурации, присвоив уникальное имя и только потом использовать фикстуру с этим именем в тесте. Было бы намного проще просто вызывать метод `create` (или подобный), который вызывал бы фикстуру, если она есть,  если нет - создавал бы её на основе фабрики.

Посему я состряпал небольшое решение этой проблемы в виде файла, который нужно сохранить в папке `support` и не забыть подключить в `spec_helper`. Выглядит он следующим образом [fbuilder](https://gist.github.com/anykey007/a681e3e8fbe7fdcd2965)

Работает он следующим образом:
* Переданные параметры хешируются (можно просто привести их к строке, т.к. сама функция хеширования нехило напрягает процессор). Хеш будет служить именем фикстуры.
* Пытаемся найти фикстуру и если находим - просто возвращаем этот объект.
* Если не находим - создаем фабрику и на основе атрибутов созданного объекто создаем фикстуру и записываем её в файл.

Пока что этот код умеет создавать фикстуры с простых фабрик. Но его можно дополнить созданием фикстур из фабрик, в которых используются колбеки.
Так же нужно решить проблему отслеживани актуальности созданных фикстур. Если фабрика изменяется, мы должны удалить и заново создать все фикстуры, которые были созданы на её основе. Для этого можно использовать хеширование файлов фабрик (как это сделали создатели [fixture_builder](https://github.com/rdy/fixture_builder)).

Все три способа можно и необходимо применять вместе. Надеюсь мой обзор вам пригодится и поможет сделать процесс тестирования таким же интересным как и непосредственно программировние.
