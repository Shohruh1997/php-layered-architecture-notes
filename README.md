<div align="center">

# 🏛️ Слоистая архитектура в PHP

**Domain · Application · Infrastructure · Interfaces** — границы, которые держат

[![PHP](https://img.shields.io/badge/PHP-8.3-777BB4?style=for-the-badge&logo=php&logoColor=white)](https://php.net)
![Доменов](https://img.shields.io/badge/доменов-11-8b5cf6?style=flat-square)
![Правило](https://img.shields.io/badge/зависимости-только_внутрь-22c55e?style=flat-square)

</div>

---

## 🗂️ Структура

```
src/
├── Domain/           бизнес-правила и контракты. Не знает ни о чём внешнем
├── Application/      сценарии использования: «принять товар», «провести документ»
├── Infrastructure/   реализации контрактов: БД, платёжные шлюзы, внешние API
├── Interfaces/       вход: HTTP-контроллеры
├── Support/          автозагрузка, конфиг, окружение
└── Legacy/           старый код, изолированный и постепенно вытесняемый
```

Домены внутри: `Catalog`, `Inventory`, `Finance`, `Contragent`, `Payment`,
`Identity`, `OnlineOrder`, `PurchaseOrder`, `Consignment`, `Integration`,
`Analytics`.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'ui-sans-serif,-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif','fontSize':'15px','primaryColor':'#4338ca','primaryTextColor':'#ffffff','primaryBorderColor':'#c7d2fe','secondaryColor':'#6d28d9','tertiaryColor':'#312e81','lineColor':'#c7d2fe','textColor':'#ffffff','mainBkg':'#4338ca','nodeBorder':'#c7d2fe','nodeTextColor':'#ffffff','edgeLabelBackground':'#1e1b4b','attributeBackgroundColorOdd':'#4338ca','attributeBackgroundColorEven':'#4f46e5','noteBkgColor':'#fbbf24','noteTextColor':'#1a1a1a','noteBorderColor':'#b45309','clusterBkg':'#1e1b4b','clusterBorder':'#c7d2fe','labelBoxBkgColor':'#4338ca','labelBoxBorderColor':'#c7d2fe','labelTextColor':'#ffffff','actorBkg':'#4338ca','actorBorder':'#c7d2fe','actorTextColor':'#ffffff','actorLineColor':'#c7d2fe','signalColor':'#c7d2fe','signalTextColor':'#ffffff','sequenceNumberColor':'#1a1a1a','activationBkgColor':'#6d28d9','activationBorderColor':'#c7d2fe','transitionColor':'#c7d2fe','transitionLabelColor':'#ffffff','stateBkg':'#4338ca','stateLabelColor':'#ffffff','altBackground':'#312e81','compositeBackground':'#1e1b4b','compositeBorder':'#c7d2fe','compositeTitleBackground':'#312e81','specialStateColor':'#c7d2fe','innerEndBackground':'#c7d2fe','cScale0':'#4338ca'}}}%%
flowchart TD
    I[Interfaces<br/>HTTP-контроллеры] --> A[Application<br/>сценарии]
    A --> D[Domain<br/>правила и интерфейсы]
    INF[Infrastructure<br/>MySQL, Payme, Click] -.реализует.-> D
    A -.получает через DI.-> INF

    style D fill:#6366f1,stroke:#4338ca,stroke-width:3px,color:#ffffff
    style INF fill:#f59e0b,stroke:#b45309,color:#1a1a1a
```

**Единственное правило, из которого следует всё остальное:** стрелки ведут внутрь.
`Domain` не импортирует ничего из `Infrastructure`. Инфраструктура зависит от
домена, а не наоборот.

---

## ❓ Зачем это, если можно писать в контроллере

Пока система маленькая, разницы нет. Она появляется в трёх местах.

**Первое: у логики становится больше одного входа.** В боевой системе «провести
документ» вызывается из веб-панели, из Telegram-бота и из фонового обработчика.
Если логика живёт в контроллере, у вас три её копии, которые расходятся.

**Второе: внешний сервис меняется.** У платёжного провайдера поменялся формат
callback. Если шлюз за интерфейсом — правка в одном классе. Если он размазан по
контроллерам — правка везде и тестирование всего.

**Третье: тесты.** Доменную логику можно проверить без базы, без сети и без
провайдера — потому что она их не знает.

---

## 🔌 Контракт в домене, реализация в инфраструктуре

Классический пример — платёжный шлюз. Интерфейс лежит в `Domain/Payment`:

```php
namespace App\Domain\Payment;

interface PaymentGateway
{
    public function providerName(): string;
    public function buildCheckoutUrl(array $order, int $paymentTransactionId): string;
    public function handleWebhook(string $rawBody, array $headers): array;
}
```

Реализации — в `Infrastructure/Payment`: `PaymeGateway`, `ClickGateway`. Домен о
них не знает; он знает только контракт.

Добавление третьего провайдера — новый класс, реализующий тот же интерфейс. Ни
одной правки в домене и сценариях. Если для нового провайдера пришлось менять
контракт — это сигнал, что абстракция протекает, и обсуждать надо её, а не
провайдера.

---

## ⚙️ Сценарий как единица работы

`Application` — это то, что происходит, а не то, как оно хранится. Один сценарий —
одна транзакция, одна ответственность.

```php
namespace App\Application\Inventory\UseCase;

final class PostInventoryDocument
{
    public function __construct(
        private DocumentRepository $documents,   // интерфейс из Domain
        private MovementRepository $movements,   // интерфейс из Domain
        private BalanceUpdater $balances,        // интерфейс из Domain
    ) {}

    public function execute(int $documentId, int $userId): void
    {
        $document = $this->documents->findDraftOrFail($documentId);

        // Движения и пересчёт остатка — в одной транзакции.
        // Иначе при сбое между ними остаток разойдётся с историей,
        // и расхождение будет тихим: ошибки нет, цифры неверные.
        $this->documents->transactional(function () use ($document, $userId): void {
            foreach ($document->items() as $item) {
                $movement = $this->movements->append($document, $item);
                $this->balances->apply($movement);
            }
            $this->documents->markPosted($document, $userId);
        });
    }
}
```

Обратите внимание: в конструкторе — интерфейсы, а не `PDO` и не конкретные классы.
Сценарий не знает, MySQL под ним или что-то другое.

---

## 🧱 Legacy как отдельный слой, а не как «потом перепишем»

`src/Legacy` — сознательное решение. Старый код не удалён и не переписан разом, он
**огорожен**: новый код в него не заглядывает, а старые вызовы постепенно
переезжают на сценарии.

Это честнее двух популярных альтернатив: «перепишем всё с нуля» (никогда не
заканчивается) и «постепенно поправим на месте» (старое расползается по новому).

Граница видна в структуре папок, а значит нарушение границы видно на код-ревью.

---

## ⚠️ Где границу нарушают чаще всего

Три типичных протечки, за которыми стоит следить:

1. **Модель БД, просочившаяся в домен.** Если доменный объект знает про
   `AUTO_INCREMENT`, `JSON`-колонку или имя таблицы — граница уже нарушена.
2. **Сценарий, собирающий HTTP-ответ.** Форматирование ответа — работа
   `Interfaces`. Сценарий возвращает данные, а не JSON со статус-кодом.
3. **Контроллер с бизнес-условиями.** Как только в контроллере появляется
   «если статус draft и роль менеджер» — это уехало из домена.

---

## 💥 Автозагрузка: ловушка регистра, стоившая продакшена

Реальная история из соседнего проекта, прямо про границы и дисциплину.

Файл назывался `src/database.php` со строчной буквы, а класс внутри —
`App\Database`. PSR-4 ищет файл `src/Database.php` с заглавной.

На Windows файловая система нечувствительна к регистру, поэтому локально всё
работало годами. На Linux — а это любой хостинг — **все страницы админки упали**
с `Class "App\Database" not found`. Уцелели только те файлы, где подключение было
явным, через `require_once`.

Второй такой же случай в том же проекте: класс числился в `autoload.files`, куда
попадают только файлы с функциями и побочными эффектами, а не классы.

Вывод, который дороже правила: **проверять проект на регистрозависимой файловой
системе до деплоя**, а не после. Одна команда в Docker-контейнере ловит это за
минуту.

---

## 🎤 Что спрашивать у кандидата по этой теме

Если вы читаете это как нанимающий — вот вопросы, которые отличают понимание от
пересказа:

- Почему интерфейс репозитория лежит в домене, а реализация в инфраструктуре?
- Что произойдёт, если пересчёт остатка вынести из транзакции с движением?
- Когда слои — лишняя сложность, а не польза?
- Как вы огораживаете legacy, если переписать нельзя?

---

## 🔗 Смежные заметки

- [db-schema-notes](https://github.com/Shohruh1997/db-schema-notes) — схемы БД этих же систем
- [inventory-accounting-notes](https://github.com/Shohruh1997/inventory-accounting-notes) — доменная логика склада
- [payment-integration-notes](https://github.com/Shohruh1997/payment-integration-notes) — шлюзы за интерфейсом

---

<div align="center">

**Шохрух Рузиев** · backend-разработчик, Ташкент

[![Сайт](https://img.shields.io/badge/ecomdev.uz-6366f1?style=for-the-badge&logo=googlechrome&logoColor=white)](https://ecomdev.uz)
[![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/EcomDev_uz)

Исходный код систем — в приватных репозиториях, доступ по запросу.

</div>
