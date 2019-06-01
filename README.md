Hedera Hashgraph - распределённый реестр, не являющийся блокчейном. База данных такого реестра строится не в виде цепочек, а в виде ацикличного направленного графа (об этом можно почитать в предыдущей статье <ссылка>). 
Однако не смотря на отличие в строении от привычных нам Блокчейн-платформ, Hedera Hashgraph включает в себя файловый сервис, смарт-контракты и криптовалюты.

<Рисунок 1>

Основной криптовалютой, работающей в сети, являются HBar'ы. Деноминация Hbar приведена для ознакомления ниже:

gigabar 1 Gℏ = 1,000,000,000 ℏ
megabar 1 Mℏ = 1,000,000 ℏ
kilobar 1 kℏ = 1,000 ℏ
hbar 1 ℏ = 1 ℏ
millibar 1,000 mℏ = 1 ℏ
microbar 1,000,000 μℏ = 1 ℏ
tinybar 100,000,000 tℏ = 1 ℏ

Меня с самого начала проект заинтересовал именно как платформа для DApp. Backend-составляющая децентрализованных приложений должна находиться внутри децентрализованной пиринговой сети. Логическая часть приложения, конкретно сам исполняемый программный код описывается с помощью смарт-контрактов.

<Схема 1>

Взаимодействие со смарт-контрактами происходит через API платформы (например, как web3 для Ethereum). Рассматриваемая нами платформа имеет собственный API, который описан в их официальном репозитории на GitHub. 
На момент мая 2019 года разработчикам ещё недоступны ноды (mirror nodes) сети, проходит вторая стадия тестирования сообществом. Взаимодействие по сети с существующими подсетями платформы осуществляется с помощью google protobuf. Для разработки DApp существуют SDK на языках Java (поддерживается официально), C, Rust, Python, GO  (поддерживаются сообществом).
Для смарт-контрактов Hedera используется язык Solidity. Это значительно упрощает миграцию на платформу.
Далее хочу описать собственный опыт взаимодействия с платформой и подводные камни, который встретились мне в процессе разработки.

Итак, я задался целью разработать своё первое приложение на Hedera Hashgraph.
Моим проектом будет криптовалютный обменник, в котором пользователи смогут обмениваться ERC-20 токенами и Hbarами. Такое приложение несложно в реализации на стороне смарт-контрактов, просто обозначим функционал и алгоритм в целом.

В нашем обменнике пользователь сможет:
1) Создать одрер на обмен одной криптовалюты на другую;
2) Исполнить уже существующий одрер;
3) Отменить собственный ордер.

У пользователя для работы с обменником будет собственный кошёлёк. Таким образом, сюда добавятся функции:
4) Ввод ERC20/Hbar-токенов на личный счёт обменника;
5) Вывод ERC20/Hbar-токенов из личного счёта.

Для хранения счёта пользователя у нас будет mapping с ключами [userAddress][token]. Для взаимодействия с криптовалютой (пункты 4,5) будут реализованы соответсвующие методы ввода/вывода. 
Логика взаимодействия с ордерами также нетривиальна: пользователь вызывает функцию создания ордера, передавая (tokenGet, amountGet, tokenGive, amountGive) в метод. Для исполнения ордера необходимо указать уникальное значение ордера либо через заранее сгенерированный hash по его полям, либо через позицию в массиве (для простоты эксперимента я взял последний вариант, однако рекомендую первый).

Отлично, код смарт-контракта написан и протестирован. Как задеплоить контракт в Hedera Hashgraph?
Воспользуемся Hedera SDK for Rust. Разберём подробно последовательность действий для деплоя SC:
1) Сгенерировать ABI для вызова методов и байткод контракта. Для разработки SC я использую Truffle, для генерации выполняю команду truffle build.
2) Загрузить байт-код контракта на платформу HH как .bin файл (см. create_file_from_file.rs). ВАЖНО! У файлового сервиса Hedera Hashgraph существует ограничение веса файлов в 6кб (ранее было заявлено 4кб, на практике оказалось другое значение). Учтите данное ограничение, очень часто вес байткода получается больше заданного лимита. В таком случае платформа позволяет загружать файл по частям (см. append_file.rs).
В случае успеха ответом сервиса будет номер файла (например, 0.0.1035).
3) Создать смарт-контракт, ссылаясь на номер файла с байткодом, уже находящегося в сети (см. create_contract.rs в ветке репозитория example/create_run_contract). В случае успеха возвращается номер смарт-контракта (например, 0.0.1536).

В дальнейшем при вызове методов SC ссылаться будем именно на его номер. Вызов функций можно изучить на примере call_hello_world_contract.rs в ветке example/create_run_contract. В транзакции помимо данных аккаунта и ноды указывается число передаваемых hbar, имя вызываемой функции, её аргументы.

Вот как выглядит фроентенд обменника, созданный на скорую руку 

(скрин)
