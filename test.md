

##### Содержание
1. IPSEC
	1. Crypto Keyring
		1. Примеры реализации
			1. Универсальный Keyring (Wildcard)
			2. Keyring с индивидуальными ключами для каждого Spoke
			3. Keyring с public IP диапазоном
		2. Особенности использования
			1. Привязка к ISAKMP Profile
			2. match identity address
		3. *Best Practices*
	2. Crypto ISAKMP Policy 
		1. Описание каждого параметра
			1. Priority
			2. Encryption Algorithm
			3. Hash Algorithm
			4. Authentication Method
			5. Diffie-Hellman Group
			6. Lifetime
		2. Особенности использования
			1. Multiple Policies
			2. Order of Preference
			3. Peer Selection
		3. *Best Practices*
	3. Crypto IPSec Transform-set
		1. Примеры реализации
			1. ESP с шифрованием AES-256 и HMAC SHA-256
			2. ESP с AES-GCM
		2. Описание протоколов и алгоритмов
			1. ESP (Encapsulating Security Payload)
			2. AH (Authentication Header)
		3. Как transform-set работает
		4. *Best Practices*
		5. Особенности использования
			1. Ordering Matters
			2. Multiple Transform-Sets
			3. Compatibility Check
			4. Заключение
	4. **Crypto IPSec Profile**
		1. Назначение
		2. Общая структура команды
		3. Описание параметров
			1. set transform-set
			2. set isakmp-profile
			3. set pfs
			4. set security-association lifetime seconds
		4. Особенности использования
		5. Best Practices
		6. Заключение
	5. **Описание функций ISAKMP**
		1. crypto isakmp invalid-spi-recovery
		2. crypto isakmp keepalive 10
		3. crypto isakmp nat keepalive 10
	6. **Shared в ISAKMP/IKE profile**
	7. 
	8. Настройка DMVPN

##### Crypto Keyring

 Назначение

crypto keyring используется для хранения параметров peer:
- IP адрес (или диапазон IP),
- pre-shared key (PSK) для аутентификации.

Keyring применяется в связке с ISAKMP profile для настройки phase 1 (IKE SA) между peer. Особенно актуален при множестве spokes в DMVPN топологии.

`crypto keyring <name>`
  `pre-shared-key address <ip> <mask> key <pre-shared-key>`

Параметры:
`<name>` – имя keyring.
`<address>` – IP адрес peer или диапазон.
`<pre-shared-key>` – ключ аутентификации.

Примеры реализации

###### 1. Универсальный Keyring (Wildcard)

Применяется, если все spokes используют одинаковый PSK.

`crypto keyring DMVPN-KEYRING`
  `pre-shared-key address 0.0.0.0 0.0.0.0 key YOUR_SECRET_KEY`

 Применение:
- Одинаковый ключ для всех peers.
- Простая DMVPN архитектура без индивидуальных security policy.

###### 2. Keyring с индивидуальными ключами для каждого Spoke

Позволяет назначать уникальный PSK каждому spoke для повышения уровня безопасности.

`crypto keyring DMVPN-KEYRING`
  `pre-shared-key address 198.51.100.10 255.255.255.255 key SPOKE1_KEY`
  `pre-shared-key address 198.51.100.11 255.255.255.255 key SPOKE2_KEY`

  Применение:
- Compliance policy (политики информационной безопасности организации) требуют уникальных PSK per peer, чтобы при компрометации ключа одного Spoke доступ к DMVPN не был возможен с других устройств.
- При аудите безопасности часто предписывается уникальность PSK для изоляции точек отказа.

###### 3. Keyring с public IP диапазоном

Используется, если Spokes имеют public IP из одного диапазона.

###### crypto keyring DMVPN-KEYRING
  ###### pre-shared-key address 203.0.113.0 255.255.255.0 key SITE_A_KEY

Применение:

- Несколько spokes, находящихся в одной площадке (например, филиал с /24 блоком IP).
- Один PSK на весь диапазон.

###### Особенности использования

###### 1. Привязка к ISAKMP Profile

Keyring активируется только при привязке к ISAKMP profile:

`crypto isakmp profile DMVPN-ISAKMP`
  `keyring DMVPN-KEYRING`
  `match identity address 0.0.0.0`

###### 2. match identity address

Определяет, каким peers применяется данный ISAKMP profile и связанный keyring. Например, wildcard (0.0.0.0) означает применение ко всем peers.

Compliance Policy и PKI
Compliance policy требует уникальных ключей
В организациях с жесткими политиками безопасности (compliance), такими как PCI DSS, ISO 27001, требования часто включают:

- Уникальные PSK для каждого peer.
- Исключение использования одного PSK на всех peers, чтобы избежать компрометации всей DMVPN инфраструктуры при утечке одного ключа.

Используется централизованная CA + PKI (для fallback на PSK per peer)

- Централизованная CA (Certificate Authority) + PKI (Public Key Infrastructure) – инфраструктура открытых ключей, позволяющая аутентифицировать peers через цифровые сертификаты, вместо PSK.
- Fallback на PSK per peer – в случае, если peer не поддерживает сертификаты (например, старое оборудование), используется fallback (резервный) вариант – PSK, при этом для каждого peer назначается уникальный PSK.

*Best Practices*

- Использовать уникальные PSK per Spoke, если это возможно.
- Wildcard keyring допустим только при строгой изоляции DMVPN от публичного доступа.
- При большом количестве spokes автоматизировать генерацию keyring entries средствами конфигурационного менеджмента (Ansible, Netmiko, Nornir).

##### Crypto ISAKMP Policy 

  

Назначение

crypto isakmp policy определяет параметры IKE Phase 1 (Internet Key Exchange), которые используются для установления ISAKMP SA (Security Association) между peers.

Это первый этап создания защищённого IPSec туннеля, в котором стороны договариваются о способе аутентификации, методах шифрования и прочих параметрах.

`crypto isakmp policy <priority>`
 `encr <encryption algorithm>`
 `hash <hash algorithm>`
 `authentication <method>`
 `group <Diffie-Hellman group>`
 `lifetime <seconds>`

Параметры:

- priority – порядковый номер policy (чем ниже, тем выше приоритет).
- encr – алгоритм шифрования (например, aes, aes 256, 3des).
- hash – алгоритм хеширования (например, sha, sha256).
- authentication – метод аутентификации (pre-share, rsa-sig, rsa-encr).
- group – Diffie-Hellman group для обмена ключами (например, 2, 14, 19).
- lifetime – время жизни SA (по умолчанию 86400 секунд).

 Пример реализации

`crypto isakmp policy 10`
 `encr aes 256`
 `hash sha256`
 `authentication pre-share`
 `group 14`
 `lifetime 86400`

 ###### Описание каждого параметра

 ###### 1. Priority

- Определяет порядок выбора политики.
- При установке IKE SA сначала предлагается policy с наименьшим номером.
- Если peer не поддерживает параметры в данной policy, пробуется следующая по приоритету.

Описание каждого параметра

 ###### 2. Encryption Algorithm

encr 

Определяет алгоритм шифрования для phase 1. Примеры:
- 3des – тройной DES, устаревающий.
- aes – AES-128.
- aes 256 – AES-256, рекомендуется к использованию.
- aes-gcm – AES в режиме GCM (при поддержке peer и IOS), обеспечивает одновременно шифрование и аутентификацию.

 ###### 3. Hash Algorithm

hash 

Применяется для проверки целостности сообщений IKE SA.
- sha или sha1 – SHA-1 (устаревающий, но ещё используется).
- sha256 – SHA-256, рекомендуется.
- md5 – устаревший, не рекомендуется.

 ###### 4. Authentication Method

authentication 

Определяет, как peer аутентифицируют друг друга:
- pre-share – pre-shared key.
- rsa-sig – цифровая подпись с использованием сертификата.
- rsa-encr – шифрование RSA, редко применяется.

 ###### 5. Diffie-Hellman Group

group 

Определяет группу DH, используемую для генерации shared secret:

| Группа | Размер   | Тип  | Применение                             |
| ------ | -------- | ---- | -------------------------------------- |
| 2      | 1024 bit | MODP | Устаревающий, минимальная безопасность |
| 14     | 2048 bit | MODP | Рекомендуется                          |
| 19     | 256 bit  | ECP  | Elliptic Curve, высокая безопасность   |
| 20     | 384 bit  | ECP  | Elliptic Curve, высокая безопасность   |
Чем выше номер группы (при ECP), тем выше стойкость.

MODP (Modular Exponential Group) – классическая DH группа на основе модульной арифметики.
ECP (Elliptic Curve Group) – группа на основе криптографии эллиптических кривых (Elliptic Curve Cryptography, ECC).

ECP использует более современные методы криптографии и тем самым используя меньший по размеру ключ более стойкая в сравнении с MODP, но может не поддерживаться на старом оборудовании.

 ###### 6. Lifetime

lifetime 

Определяет, как долго SA остаётся активной, после чего инициируется renegotiation. По умолчанию 86400 секунд (24 часа).

При работе с разными вендорами рекомендуется согласовать lifetime для предотвращения неожиданных разрывов.

 Особенности использования

###### 1. Multiple Policies

Можно настроить несколько ISAKMP policies с разными параметрами и приоритетами для обеспечения совместимости с различными peers.

###### 2. Order of Preference

При инициировании туннеля маршрутизатор предлагает policies в порядке возрастания приоритета (наименьшее число первым).

###### 3. Peer Selection

Peer выбирает первую policy, параметры которой совпадают с его собственной конфигурацией.

 *Best Practices*

- Использовать AES-256 и SHA-256 как baseline для новых туннелей.
- Применять Diffie-Hellman group 14 или выше.
- Минимизировать использование устаревших алгоритмов (3DES, MD5, DH group 2).
- В много-вендорных средах проверить совместимость настроек с оборудованием всех производителей.
- Lifetime: оставить default (86400) или сократить до 28800, если это policy организации.

 Compliance и security requirements

Многие стандарты (например, PCI DSS, ISO 27001) требуют использования:
- Алгоритмов AES (128/256),
- SHA-2 family (sha256/384/512),
- Diffie-Hellman group не ниже 14 (2048 bit) или Elliptic Curve (group 19/20) для обеспечения криптостойкости.

*Заключение*

crypto isakmp policy – критическая часть настройки IPSec VPN, определяющая phase 1 security parameters, которые должны совпадать между peers для успешной и безопасной установки SA.

 ##### **Crypto IPSec Transform-set** 

 Назначение

crypto ipsec transform-set определяет набор протоколов и алгоритмов, которые используются для защиты данных в IPSec Phase 2 (Quick Mode).

Это комбинация:

- ESP (Encapsulating Security Payload) или AH (Authentication Header)
- Алгоритмов шифрования (encryption)
- Алгоритмов аутентификации (integrity)

 Общая структура команды

`crypto ipsec transform-set <name> <transform1> [transform2] [transform3]`

 Параметры:

 `<name>` – имя transform-set.
 `<transform>` – протокол(ы) и алгоритм(ы) IPSec защиты.

 Примеры реализации

 ###### 1. ESP с шифрованием AES-256 и HMAC SHA-256

`crypto ipsec transform-set DMVPN-SET esp-aes 256 esp-sha256-hmac`

- esp-aes 256 – шифрование с AES-256.
- esp-sha256-hmac – хеширование и аутентификация с SHA-256 HMAC.

 ###### 2. ESP с AES-GCM

`crypto ipsec transform-set GCM-SET esp-gcm 256`

- esp-gcm 256 – режим Galois/Counter Mode, объединяющий шифрование и аутентификацию в одном алгоритме.

 3. AH без шифрования (только аутентификация)

`crypto ipsec transform-set AUTH-ONLY ah-sha-hmac`

- ah-sha-hmac – AH protocol с HMAC SHA-1.
- Применяется редко, так как AH не шифрует данные, только обеспечивает аутентификацию заголовков и полезной нагрузки.

 **Описание протоколов и алгоритмов**

 ESP (Encapsulating Security Payload)

- Используется для шифрования и/или аутентификации данных.
- Поддерживает как конфиденциальность (encryption), так и целостность (integrity).
- Алгоритмы:
    - esp-aes, esp-aes 256 – AES шифрование.
    - esp-3des – тройной DES, устаревающий.
    - esp-gcm – AES Galois/Counter Mode (шифрование + аутентификация).
    - esp-sha-hmac, esp-sha256-hmac – хеширование SHA-1 или SHA-256.

AH (Authentication Header)

- Обеспечивает аутентификацию и целостность, но не шифрует данные.
- Используется крайне редко в современных VPN из-за отсутствия конфиденциальности.

 **Как transform-set работает**

1. При установке IPSec SA оба peer договариваются о transform-set, который поддерживается обеими сторонами.
2. Transform-set применяется в crypto ipsec profile или crypto map и связывается с tunnel interface или GRE-туннелем.
3. Определяет, какие алгоритмы будут использованы для защиты данных внутри IPSec SA.

 *Best Practices*

- Использовать ESP с AES-256 и SHA-256 HMAC как baseline.
- При наличии поддержки использовать esp-gcm для повышения производительности (шифрование и аутентификация в одном алгоритме снижает нагрузку на CPU).
- Избегать 3DES и MD5 как устаревшие алгоритмы.
- AH использовать только при специфичных требованиях (например, compliance, где запрещено шифрование, но требуется аутентификация).

 **Особенности использования**

###### 1. Ordering Matters

Порядок transform в set имеет значение. Например:
crypto ipsec transform-set EXAMPLE esp-aes esp-sha-hmac

означает, что ESP будет использовать AES для шифрования и SHA-1 HMAC для аутентификации.

###### 2. Multiple Transform-Sets

Можно создать несколько transform-set с разными наборами алгоритмов для совместимости с различными peers.
*Использование нескольких trasform-set возможно только в crypto-map, если требуется это реализовать в tunnel protection, то необходимо создать отдельный ipsec profile под peer.

###### 3. Compatibility Check

Оборудование на обеих сторонах VPN должно поддерживать одинаковые алгоритмы из transform-set.

Заключение

crypto ipsec transform-set – это определение используемых протоколов шифрования и аутентификации IPSec Phase 2.

Правильный выбор transform-set обеспечивает:

- Соответствие требованиям безопасности организации (Compliance)
- Производительность туннеля без перегрузки CPU
- Совместимость с peers (особенно в multi-vendor средах)

 ###### **Crypto IPSec Profile** 

  
 Назначение

crypto ipsec profile используется для интеграции IPSec защиты с tunnel интерфейсами (например, GRE-туннелями) при использовании команд tunnel protection ipsec profile.

Он определяет:

- какие transform-set применять,
- какой isakmp profile использовать (если необходимо),
- дополнительные политики IPSec.

 Отличие от crypto map

|Параметр|crypto ipsec profile|crypto map|
|---|---|---|
|Применение|Туннельные интерфейсы (GRE/IPSec VPN)|Policy-based VPN (site-to-site)|
|Использование|DMVPN, GRE over IPSec|Классический IPSec без туннеля|
|Привязка|tunnel protection ipsec profile|Привязывается к physical interface|

 Общая структура команды

`crypto ipsec profile <name>`
 `set transform-set <transform-set-name>`
 `set isakmp-profile <isakmp-profile-name>`
 `set pfs <group>]`
 `set security-association lifetime seconds <value>`

 Параметры:

  `name` – имя профиля.
  `set transform-set` – указание transform-set, применяемого к SA.
  `set isakmp-profile` – (опционально) связывает ipsec profile с isakmp profile для специфичной аутентификации.
  `set pfs` – Perfect Forward Secrecy, выбор DH группы для phase 2.
  `set security-association lifetime seconds`  – lifetime SA для phase 2.

 Описание параметров

 ###### 1. set transform-set

Определяет, какой transform-set применяется для шифрования и аутентификации IPSec SA.

 ###### 2. set isakmp-profile

Привязывает профиль к определенному isakmp profile, который содержит информацию об аутентификации peer (например, keyring).

Используется, если необходимо:

- Применять разные аутентификационные профили к разным peers.
- Реализовать fallback PKI/PSK 
- Настраивать специфичные policies per peer.

 ###### 3. set pfs

Perfect Forward Secrecy (PFS) – опция, требующая повторного DH exchange для каждого SA, повышая безопасность.

Например:

set pfs group14

Когда применять:

- При строгих security requirements (PCI DSS, ISO 27001).
- При работе с чувствительными данными.
- При поддержке со стороны peer.

 ###### 4. set security-association lifetime seconds

Устанавливает lifetime SA для phase 2 (по умолчанию 3600 секунд).

Пример:
set security-association lifetime seconds 28800

 Применение на Tunnel Interface

Пример использования IPSec profile в tunnel интерфейсе (например, DMVPN HUB/Spoke):

`interface Tunnel0`
`tunnel protection ipsec profile DMVPN`

 **Особенности использования**

###### 1. Без указания transform-set profile не будет работать.
###### 2. isakmp-profile обязателен, если используется:
    - Несколько keyrings per peer.
    - PKI с fallback на PSK.
    - Аутентификация по сертификатам.
###### 3. PFS увеличивает нагрузку на CPU при renegotiation SA, но обеспечивает более высокую криптостойкость.
###### 4. Lifetime SA должен быть согласован между peers для предотвращения неожиданных разрывов туннеля.

 *Best Practices*

- Всегда использовать transform-set с AES и SHA-2 family.
- Включать PFS при наличии требований compliance.
- Настраивать isakmp profile, если используется keyring с множественными pre-shared keys или PKI.
- Lifetime SA phase 2: обычно 3600–28800 секунд, в зависимости от политики организации.

*Заключение*

crypto ipsec profile – это связующее звено между GRE туннелями (или DMVPN) и IPSec, обеспечивающее:
- Применение transform-set
- Использование isakmp profile (PKI/PSK аутентификация)
- Конфигурацию PFS и SA lifetime
Без ipsec profile команда tunnel protection ipsec profile работать не будет, и туннель останется без шифрования.

 **Описание функций ISAKMP**

 ###### 1. crypto isakmp invalid-spi-recovery

 Назначение

Включает механизм автоматического удаления “stale” Security Associations (SA) в случае, если peer возвращает ошибку invalid SPI (Security Parameter Index).

 Контекст

- SPI – это уникальный идентификатор SA.
- Иногда после перезагрузки peer или переинициализации туннеля одна сторона может продолжать использовать старый SPI, который уже недействителен на другой стороне.
- В таком случае трафик сбрасывается, а SA остаётся в невалидном состоянии, блокируя переинициализацию IPSec SA.

 Как работает

При включении команды:
`crypto isakmp invalid-spi-recovery`

- Устройство автоматически удаляет невалидную SA при получении сообщения invalid SPI от peer.
- Это предотвращает зависание туннеля и устраняет необходимость ручного удаления SA (командой clear crypto sa).

 Best Practice

Рекомендуется включать во всех DMVPN/IPSec конфигурациях для автоматической очистки stale SA и повышения отказоустойчивости туннелей.

 ###### 2. crypto isakmp keepalive 10

 Назначение

Включает механизм Dead Peer Detection (DPD), который периодически проверяет доступность peer.

 Как работает
`crypto isakmp keepalive 10`

- Отправляет keepalive сообщения каждые 10 секунд.
- Если peer не отвечает в течение двух keepalive интервалов (по умолчанию), SA считается “мертвой”, и производится пересоздание туннеля.
 
 Применение

- Устраняет проблему “висячих” туннелей при silent failure peer (например, если peer недоступен, но SA сохраняется как активная).
- Ускоряет failover при использовании резервных каналов или multiple hubs в DMVPN.

 *Best Practice*
 
Рекомендуется настраивать keepalive с интервалом 10–30 секунд для WAN туннелей в зависимости от требований к convergence time.

 ###### 3. crypto isakmp nat keepalive 10

 Назначение

Активирует механизм отправки NAT keepalive сообщений при использовании NAT Traversal (NAT-T).

 Контекст

При работе IPSec через NAT устройства:

- NAT-T encapsulates ESP в UDP 4500.
- NAT устройств может иметь таймаут на UDP session (например, 30 секунд).
- Если между peer нет трафика, NAT device удаляет mapping, и IPSec туннель рвётся.

 Как работает
`crypto isakmp nat keepalive 10`

- Отправляет NAT keepalive пакеты каждые 10 секунд для поддержания UDP mapping в NAT устройстве.

 Применение

- Обязателен при прохождении IPSec через NAT (например, spoke маршрутизатор за NAT в сети провайдера).
- Предотвращает обрывы туннеля из-за таймаутов NAT таблиц.

 *Best Practice*
 
Всегда включать при работе IPSec VPN через NAT устройства (провайдерские NAT, CGNAT, корпоративные NAT).

|Команда|Назначение|Применение|
|---|---|---|
|crypto isakmp invalid-spi-recovery|Автоудаление невалидных SA (invalid SPI)|Устранение stale SA, автоматический recovery|
|crypto isakmp keepalive 10|Dead Peer Detection (DPD) с интервалом 10 сек|Проверка доступности peer, быстрый failover|
|crypto isakmp nat keepalive 10|NAT keepalive каждые 10 сек|Поддержание NAT mappings, предотвращение обрывов|

*Заключение*

Все три функции повышают стабильность IPSec туннелей, уменьшают необходимость ручного вмешательства при обрывах или silent failure peer, и рекомендуются к применению в production DMVPN/IPSec архитектурах.



### **Shared в ISAKMP/IKE profile**

**shared** – это опция, позволяющая использовать **один ISAKMP/IKE профиль одновременно в нескольких VRF**, создавая общую (shared) SA (Security Association) между VRF.

###### **Как работает ISAKMP/IKE profile без shared:**

1. **ISAKMP/IKE профиль** привязывается к определённой SA, созданной в конкретном VRF.
2. **По умолчанию** профиль не может быть использован в другом VRF, даже если конфигурация идентична, так как SA не “видна” за пределами VRF, в котором создана.
3. Попытка использовать его в нескольких VRF без shared приведёт к ошибке сопоставления ISAKMP profile.

###### **Как работает shared:**

  

Когда ты добавляешь:

`crypto isakmp profile <name> shared`

Профиль становится **“shared across VRFs”** (общим между VRF)
SA, созданная по этому профилю, может применяться в **разных VRF**
Это позволяет использовать **один ISAKMP/IKE профиль** для нескольких туннелей в разных VRF без конфигурации отдельных профилей под каждый VRF.


| **Сценарий**                                                      | **Нужно ли shared?** | **Почему**                                  |
| ----------------------------------------------------------------- | -------------------- | ------------------------------------------- |
| **1. Один и тот же ISAKMP/IKE профиль используется в разных VRF** | **Да**               | Без shared SA будет ограничена одним VRF.\| |
| **2. Разные профили в разных VRF**                                | **Нет**              | У каждого VRF своя SA.                      |
| **3. Один профиль, несколько туннелей в одном VRF**               | **Нет**              | Все туннели используют SA в одном VRF.      |
| **4. Multicast DMVPN (mGRE в разных VRF с одной SA)**             | **Да**               | Мультикаст SA должна быть общей.            |
###### **Типичные применения shared:**

**Dual VRF WAN design**
Например, у тебя VRF “WAN1” и VRF “WAN2” на одном устройстве, и оба используют один ISAKMP профиль для подключения spoke через разные провайдеры.

**VRF-lite Interconnect через IPsec**
Если один и тот же профиль используется для IPsec туннелей, которые находятся в разных VRF.

**Multicast DMVPN**
Для мультикаста необходимо, чтобы SA была общей между VRF, где реализован mGRE.


### **Shared в tunnel protection ipsec profile**

Это нужно для **DMVPN mGRE**, чтобы **SA была shared между туннелями**.

Особенно актуально, когда есть:

- Multicast DMVPN (Phase 3)
- Несколько туннелей с одинаковым IPsec profile, чтобы использовать **одну SA** (экономия CPU/RAM на хабах и споуках).

-----




#### Настройка DMVPN


`interface Tunnel0`
 `description MPLS`
 `ip address 10.252.201.253 255.255.254.0`
 `no ip redirects`
 `ip mtu 1300`
 `ip nhrp authentication Gl0Bu$`
 `ip nhrp network-id 666`
 `ip tcp adjust-mss 1280`
 `load-interval 30`
 `mpls ip`
 `mpls bgp forwarding`
 `tunnel source GigabitEthernet1.2750`
 `tunnel mode gre multipoint`
 `tunnel key 666`
 `tunnel vrf GLOBAL`
 `tunnel protection ipsec profile DMVPN shared`
`!`
`interface Tunnel1`
 `description No_MPLS`
 `ip vrf forwarding PRIVATE`
 `ip address 10.252.202.253 255.255.255.0`
 `no ip redirects`
 `ip mtu 1300`
 `ip nhrp authentication Gl0Bu$`
 `ip nhrp network-id 888`
 `ip tcp adjust-mss 1280`
 `load-interval 30`
 `tunnel source GigabitEthernet1.2750`
 `tunnel mode gre multipoint`
 `tunnel key 888`
 `tunnel vrf GLOBAL`
 `tunnel protection ipsec profile DMVPN shared`

Ключевых отличий от старой конфигурации на хабах ASR DC/LC - нет, только на данных туннелях с помощью идентификатора сети (ip nhrp network-id 888) происходит разделение сетей. На mpls в конфиге не обращаем внимание, так как в данной документации мы его не будем разбирать. 
Причина почему на одном из туннелей есть vrf forwarding, а на другом нет, как раз кроется в использовании mpls, так как не все Spok маршрутизаторы (Лавки) поддерживают mpls и что бы добиться сегментации на точках где нет mpls будет использоваться сегмантация с помощью vrf (решение временное)

В настройке туннелей на Spok маршрутизаторах отличий больше

```bash
interface Tunnel0
 ip address 10.252.200.4 255.255.254.0
 no ip redirects
 ip mtu 1300
 ip nhrp authentication Gl0Bu$
 ip nhrp map 10.252.201.254 185.11.197.94
 ip nhrp map 10.252.201.253 185.11.197.131
 ip nhrp map multicast 185.11.197.94
 ip nhrp map multicast 185.11.197.131
 ip nhrp network-id 666
 ip nhrp holdtime 60
 ip nhrp nhs 10.252.201.254
 ip nhrp nhs 10.252.201.253
 ip nhrp registration timeout 30
 ip tcp adjust-mss 1280
 load-interval 30
 mpls ip
 mpls bgp forwarding
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel key 666
 tunnel protection ipsec profile DMVPN
```

```python

async def main(sites):
    # for site in sites:
    #     print(site)
    result = []
    tasks = [asyncio.create_task(greate_site(site)) for site in sites]
    for coro in asyncio.as_completed(tasks):
        result.append(await coro)
    # for coro in asyncio.as_completed(tasks):
    #     result.append(await coro)
        # name = site['name']
        # slug = site['slug']
        # constructed_site = {"name": name, "slug": slug}
        # if "description" in site.keys():
        #     constructed_site['description'] = site['description']
        # if "physical_address" in site.keys():
        #     constructed_site['physical_address'] = site['physical_address']
        # if "slug" in site.keys():
        #     constructed_site["slug"] = site["slug"]
        # print(constructed_site)
    # await asyncio.gather()

if __name__ == "__main__":
    asyncio.run(main(sites_to_load))

```

```yaml

  - name: lighthouse
    src: git@github.com:vladrabbit/lighthouse.git
    scm: git
    version: "1.0.0"

```



Как мы видим отличие от legacy туннелей на других объектах в дополнительных команда применяемых в данном туннеле.

ЧТо добавилось:
 `ip nhrp map 10.252.201.254 185.11.197.94`
 `ip nhrp map 10.252.201.253 185.11.197.131`
 `ip nhrp map multicast 185.11.197.94`
 `ip nhrp map multicast 185.11.197.131`
 `ip nhrp nhs 10.252.201.254`
 `ip nhrp nhs 10.252.201.253`
 

В данном примере мы добавляем второй хаб, что бы Spoke мог подключаться сразу к двум держа две активные сессии, но основным хабом будет тот кто первым прописан в команда `ip nhrp nhs`. Если первый отключится, то на его место встанет второй. Оба хаба с помощью протоколов динамической маршрутизации будут обмениваться префиксами не зависимо кто primary, а кто secondary.

Для функционирования данной схемы вместо команды `tunnel destination` используется команда  `tunnel mode gre multipoint`, так как подключение идет сразу в двум хабам в отличии от стандартной legacy настройки.