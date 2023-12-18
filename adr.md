# Задача
Имеем пекарю. В рамках маленькой пекарни есть склад материалов (мука, масло ..), производственный цех, склад готовой продукции, собственная доставка. Заказов много, хотим автоматизировать процесс. Сам процесс выглядит так. Клиент определяет по каталогу продукт, делает заказ. Проверяем сколько у нас на складе ингредиентов, и вычисляем ориентировочную дату (например, если муки достаточно, то заказ выполним и доставим через 4 дня). Далее уведомляем клиента, о дате, клиент подтверждает заказ. Заказ передаем в цех, цех заказывает со склада муку и проч. и начинает изготовление. После изготовление товар отправляется на склад готовой продукции, оттуда товар отправляется клиенту. Хотим автоматизировать весь процесс - прием заказа, обработку заказа, доставку заказа. В цеху ставим электронное табло, в котором будут появляться заказы на изготовление.



# Описание системы
### Контекстная диаграмма, С1
```plantuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
HIDE_STEREOTYPE()

Person(user, "Пользователь")
Person(stock_manager, "Кладовщик")
Person(baker, "Пекарь")

Container(front, "Сайт/мобильное\nприложение")
Container(payment_service, "Экваиринг", $descr="Платежный шлюз\nбанка")
Container(system, "Система управления\nпроцессами пекарни", $descr="1. Процессит заказы\n2. Считает сроки доставки\n3. Хранит остатки, процессит\nсклад")

Rel_R(user, front, "1. Регистрируется\n2. Пользуется каталогом\n3. Делает заказы, оплачивает\n4. Следит за статусом заказа\n5. Смотрит историю заказов")
Rel_D(front, system, "Получает информацию из\nсистемы управления пекарней")
Rel_U(stock_manager, system, "Вносит информацию об остатках\nи перемещениях со склада")
Rel_U(baker, system, "Вносит изменения по\nстатусам заказов")
Rel_R(front, payment_service, "Оплата заказов")
```
### Компонентная диаграмма, C3
```plantuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
HIDE_STEREOTYPE()
AddElementTag("fallback", $bgColor="#03CD05")

Container(front, "Сайт/мобильное\nприложение")
Container(payment_service, "Экваиринг", $descr="Платежный шлюз\nбанка")
Container(nginx, "NGINX", $tags="fallback")
ContainerDb(masterDb, "masterDb")
ContainerDb(redis, "redis")
ContainerDb(cdn, "cdn хостинга", $descr="отдает контент\nдля каталога")
System_Boundary(c1, "Система управления\nпроцессами пекарни", $link="https://github.com/plantuml-stdlib/C4-PlantUML") {
    Container(order_service, "order_service")
    Container(stock_service, "stock_service")
    Container(internal_api, "internal api", "")
}

System_Boundary(c2, "Web application", $link="https://github.com/plantuml-stdlib/C4-PlantUML") {
    Container(client_api, "Клиентские api", $descr="/catalog\n/checkStock\n/makeOrder\n/myOrders")
}

Container(ui, "Web-UI системы\nуправления пекарней", $descr="Используется пекарями,\nкладовщиками")

Rel_L(ui, internal_api, "")
Rel_D(internal_api, stock_service, "")
Rel_D(internal_api, order_service, "")
Rel_L(client_api, internal_api, "Сервисные запросы\n/checkStock\n/makeOrder", "gRPC")

Rel_L(cdn, front, "")

Rel(client_api, redis, "Чтение данных для каталога")

Rel(front, nginx, "")
Rel(nginx, client_api, "")
Rel_L(client_api, payment_service, "Инициация оплаты")
Rel(redis, masterDb, "")
Rel(stock_service, masterDb, "")
Rel(order_service, masterDb, "")
```
Краткое summary по предлагаемому решению:\
Клиент заходит на сайт/приложение. Фронт обращается в /catalog, представляет пользователю список товаров, картинки. Для работы фронта используем стандартную схему с кэшированием в redis, тяжелый контент (картинки и проч.) храним в cdn хостинга.\
Клиент выбирает товар, кладет в корзину - фронт обращается на /checkStock, далее вызывается метод проверки остатков в системе управления пекарней (gRPC здесь лучше rest ввиду быстроты и отсутствия необходимости декорации получаемых данных.)\
Клиент делает заказ, уходит запрос на /makeOrder, инициируется оплата в платежном шлюзе банка-эквайера.\
Система управления пекарней на старте может состоять всего из двух сервисов (или компонентов в случае монорепы) соответственно:\
order_service - создает заказы, меняет их статусы, возвращает статусы;\
stock_service - для работы с остатками.\
Для пекарей и кладовщиков делаем web-ui для отражения операций по процессу, из него же можем выводить на электронное табло список текущих заказов и их статусы.

