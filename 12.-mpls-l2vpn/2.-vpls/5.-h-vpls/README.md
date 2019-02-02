# Иерархический VPLS \(H-VPLS\)

Изначально VPLS — это плоская технология — все соседи одного ранга. И если в Kompella mode RR в очередной раз эффективно решают проблему полносвязной топологии, то Martini Mode в операторских сетях может вызвать кризис рабочей силы — все инженеры будут только настраивать удалённые LDP-сессии. Напомню сложность задачи O\(n^2\): n\*\(n-1\)/2 — где n — число узлов.

![](https://habrastorage.org/files/485/7c3/61f/4857c361f37b43e78ac87495cca44029.PNG)

Такая топология скрывает и ещё одну проблему — широковещательные кадры Ethernet — каждый пакет будет повторён столько раз, сколько соседей у данного PE.  
А не можем ли мы здесь применить что-то вроде BGP-шных Route Reflector'ов?

Можем. Наш путь к спасению — H-VPLS \(Hierarchical VPLS\), который описан в [RFC 4762](https://tools.ietf.org/html/rfc4762).

H-VPLS разделяет маршрутизаторы VPLS-домена на два ранга: PE-rs и MTU-s.  
**PE-rs** — PE — routing and switching. Это ядро сети VPLS. Это большие производительные железки, которые функционируют как обычные PE.  
Другие названия PE-rs: [**PE-POP**](http://lookmeup.linkmeup.ru/#term596), [**n-PE**](http://lookmeup.linkmeup.ru/#term595).  
**MTU-s** — Multi-Tenant Unit — switching. Это могут быть более слабые устройства, которые подключаются к PE-rs с одной стороны. А с другой к ним подключаются CE.  
Другие названия MTU-s: [**u-PE**](http://lookmeup.linkmeup.ru/#term599), [**PE-CLE**](http://lookmeup.linkmeup.ru/#term600).

![](https://habrastorage.org/files/119/97e/d22/11997ed220d94f16bf1c7b9a81d96384.PNG)

MTU-s обычно имеют восходящие интерфейсы к двум PE-rs — для отказоустойчивости. Но может быть и один.  
Механизмы подключения MTU-s к PE-rs: MPLS PW или QinQ. Поскольку мы тут про MPLS, то будем рассматривать только первый способ.

PE-rs между собой взаимодействуют как обычные PE, образуя полносвязную топологию и используют правило расщепления горизонта.  
При взаимодействии PE-rs и MTU-s, PE-rs воспринимает PW от MTU-s как AC-интерфейс, то есть, как рядового клиента.  
А значит:  
— Всё, что PE-rs получил от MTU-s он рассылает всем соседним PE-rs и всем подключенным MTU-s,  
— То, что PE-rs получил от соседнего PE-rs он рассылает всем подключенным к нему MTU-s, но не отправляет другим PE-rs.  
Таким образом, каждому MTU-s нужно поддерживать связь только с двумя \(или одним\) соседом, а количество PE-rs достаточно невелико, чтобы организовать между ними полносвязную топологию.  
При добавлении нового узла нужно настроить только его и вышестоящий PE-rs.

Для организации Inter-AS VPN H-VPLS тоже может сослужить службу. Например, два подключенных друг к другу ASBR могут быть PE-rs более высокого уровня иерархии.

Итак H-VPLS — это решение трёх задач:

* Улучшение масштабируемости сети в плане Contol Plane.
* Оптимизация Data Plane за счёт уменьшения числа копий широковещательных кадров.
* Возможность делить VPLS-домен на сегменты и ставить на доступ более дешёвое оборудование.

Так! Стоп, а что насчёт MAC-таблицы на PE-rs?  
То, что в обычном VPLS было P, в H-VPLS стало PE, а соответственно, должно заниматься клиентскими данными — изучать MAC-адреса. Причём от всех MTU-s, которые к нему подключены. А вместе с тем заниматься рассылкой и репликацией клиентских кадров.  
Получается, что, введя иерархию на Control Plane мы форсировали создание иерархии и на Data Plane. Справившись с одной проблемой масштабируемости, H-VPLS создал новую. Счёт MAC-адресов в этом случае может идти на тысячи и сотни тысяч, вопрос флудинга встаёт на новом уровне, нагрузка на CPU может значительно возрасти  
Но решения для этой ситуации H-VPLS не предлагает.  
Впрочем, удешевление устройств уровня доступа, видимо, вполне окупает этот _лёгкий_ дискомфорт.

Ну что, попробуем настроить?