# Lab028 Управление анонсами BGP

# Исходное домашнее задание:
- внешний вид сети(рассматриваемый в данной лабе фрагмент):
![start](start.png)

## Поставленные задачи
1. Настроить фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика(As-path).
2. Настроить фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика(Prefix-list).
3. Настроить провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.
4. Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.
5. Все сети в лабораторной работе должны иметь IP связность.
6. План работы и изменения зафиксированы в документации.

### Выполнение задания
####  Настраиваем фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика(As-path)
На R14: 
```
ip as-path access-list 1 permit ^$
```
Создаём правило в которое попадают только те маршруты, которые не имеют транзитных AS-path.
Теперь применяем его:
```
router bgp 1001
neighbor 172.16.0.2 filter-list 1 out
```
Т.е.фильтруем все транзитные маршруты в сторону Киторна.
Смотрим на маршруты в Киторне:
```
R22#sho ip bgp neighbors 172.16.0.1 routes 
BGP table version is 78, local router ID is 10.1.0.22
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.0.1.0/24      172.16.0.1               0             0 1001 i
 *>  10.0.2.0/24      172.16.0.1               0             0 1001 i
 *   10.3.0.21/32     172.16.0.1                             0 1001 301 i
 *   10.5.2.0/24      172.16.0.1                             0 1001 301 520 i
 *   10.20.42.0/24    172.16.0.1                             0 1001 301 520 2042 i
 *>  192.168.1.0      172.16.0.1             101             0 1001 i
 *   192.168.42.0     172.16.0.1                             0 1001 301 520 2042 i
```
Очевидно, что наши изменения еще не применились.  Запускаем (на R14):
```
clear ip bgp * soft
```
Смотрим результат:
```
R22#sho ip bgp neighbors 172.16.0.1 routes 
BGP table version is 78, local router ID is 10.1.0.22
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.0.1.0/24      172.16.0.1               0             0 1001 i
 *>  10.0.2.0/24      172.16.0.1               0             0 1001 i
 *>  192.168.1.0      172.16.0.1             101             0 1001 i
```
Видим, что после bgp update наш роутер в Киторне получает только маршруты из as 1001, а транзитные маршруты исключены.
Теперь то же самое проделаем на роутере R15:

```
ip as-path access-list 1 permit ^$
router bgp 1001
neighbor 172.16.0.14 filter-list 1 out 

clear ip bgp * soft
```
Итого, маршруты на Ламасе:
```
R21#sho ip bgp neighbors 172.16.0.13 routes
BGP table version is 46, local router ID is 10.3.0.21
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.0.1.0/24      172.16.0.13              0             0 1001 i
 *>  10.0.2.0/24      172.16.0.13              0             0 1001 i
 *>  192.168.1.0      172.16.0.13            311             0 1001 i

Total number of prefixes 3 
```
Задача выполнена.
#### Настроика фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика(Prefix-list).
Чтобы решить эту задачу, нужно сделать так, чтобы роутер  R18 анонсил только свои сети.
Поехали:
```
ip as-path access-list 1 permit ^$
router bgp 2042
  neighbor 172.16.0.21 filter-list 1 out
  neighbor 172.16.0.25 filter-list 1 out
```
т. е. теперь из офиса Питера мы анонсим только свои сети, поэтому транзитный трафик через нее не пойдёт.
UPDATE: 19.09.2023
 теперь явно зафильтруем префиксы, которые отдаём наружу,в провайдер "Триада".
На R18:
```
ip prefix-list outTriada permit 192.168.42.0/24
ip prefix-list outTriada permit 10.20.42.0/24
```
А также применим этот prefix-list на соседей:
```
neighbor 172.16.0.21 prefix-list outTriada out
neighbor 172.16.0.25 prefix-list outTriada out
```
Готово. Теперь наружу кроме явно указанных двух сеток больше ничего не уйдёт.

#### Настроика провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.
Исходные данные: Изначально мы получаем от Киторна вот такие маршруты:
```
sho ip bgp nei 172.16.0.2 route
BGP table version is 117, local router ID is 10.0.1.14
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.1.0.22/32     172.16.0.2               0             0 101 i
 *   10.3.0.21/32     172.16.0.2                             0 101 301 i
 *   10.5.2.0/24      172.16.0.2                             0 101 520 i
 *   10.20.42.0/24    172.16.0.2                             0 101 520 2042 i
 *>  172.16.0.4/30    172.16.0.2               0             0 101 i
 *>  172.16.0.8/30    172.16.0.2               0             0 101 i
 *   192.168.42.0     172.16.0.2                             0 101 520 2042 i
```
начинаем переделывать:
Сначала добавим в Киторне (R22) маршрут по умолчанию:
```
ip route 0.0.0.0 0.0.0.0 Null 0
```
Теперь добавим его в анонсы:
```
router bgp 101
   network 0.0.0.0 mask 0.0.0.0
```
Ну и теперь остаётся только зафильтовать (на R22):
```
ip prefix-list 1 permit 0.0.0.0/0
router bgp 101
  neighbor 172.16.0.1 prefix-list 1 out 

clear ip bgp * soft
```
И теперь проверяем маршрутную информацию на R14:
```
R14#sho ip bgp nei 172.16.0.2 route
BGP table version is 124, local router ID is 10.0.1.14
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          172.16.0.2               0             0 101 i

```
Готово. Задача выполнена.
UPDATE: 19.09.2023. Переделаю механизм анонса маршрута по умолчанию на neighbor ... default-originate
На R22:
```
no ip route 0.0.0.0 0.0.0.0 Null 0

router bgp 101
no network 0.0.0.0 mask 0.0.0.0
 neighbor 172.16.0.1 default-originate

```
Готово. Задача выполнена.

####  Настроика провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург

Немного подумав, решил решить эту задачу с помощью инструмента route-map.
Сделав в нём 2 правила:
1. Все тот же as-path filtering
2. фильтрация по prefix list.
Затем, объеденим сие в 1 route-map и применим на соседа:

На R21:
```
ip as-path access-list 2 permit _2042$

ip prefix-list 1 seq 5 permit 0.0.0.0/0
```
Правила созданы, теперь  их нужно "склеить" в 1 правидо route-map:
```
route-map toMoscow permit 10
 match as-path 2

route-map toMoscow permit 20
 match ip address prefix-list 1
```
И навесить на соседа:
```
 neighbor 172.16.0.13 route-map toMoscow out
```
Взбодрим систему посредством:
```
clear ip bgp * soft
```
И можно идти смотреть на R15 (В Москве), что у нас получилось в итоге:

```
R15#sho ip bgp nei 172.16.0.14 rou
BGP table version is 160, local router ID is 10.0.1.15
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          172.16.0.14              0             0 301 i
 *>  10.20.42.0/24    172.16.0.14                            0 301 520 2042 i
 *>  192.168.42.0     172.16.0.14                            0 301 520 2042 i
```
Как видим ,условия задачи - выполнены: мы получаем от "Ламас":
default и сети офиса С. Петербург.

#### Конфигурации оборудования.
Готовые конфигурации оборудования были экспортированы в папку configs (zip архив, выгруженный из PnetLab. Так можно?)


