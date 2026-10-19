# Multi-turn Test Plan + Degradation Profile — шаблон

**Додатковий матеріал до заняття 5 «Context failure у діалозі».**

Це робоча форма на всю тему. Ви почнете заповнювати її на лабораторній, а в ДЗ_4 доповните й здасте. Біля кожного розділу вказано, коли його заповнювати і де в темі розібрано, що туди писати. Після курсу той самий шаблон підходить для діалогової системи вашої команди.

## Як користуватися

1. Перед лабораторною скопіюйте шаблон собі й назвіть `multi-turn-test-plan.md`. На лабораторній записуйте результати прямо в нього.
2. У ДЗ_4 покладіть цей самий файл у репозиторій і допишіть розділи з позначкою «ДЗ». Числа з лабораторної можна лишити, якщо в них є входи кожного кроку.
3. Під кожним розділом є рядок «Заповнюєте»:
    - **Лабораторна** — заповнюєте під час відповідного кроку лабораторної;
    - **ДЗ** — заповнюєте вдома, у названому кроці ДЗ_4.
4. Рядки-підказки курсивом після заповнення можна видалити.

## Що робить цей документ документом рішення

- кожне твердження про механізм збою спирається на **входи кроку**: історію і retrieval-контекст; текст відповіді доказом не вважається;
- крок втрати подано **розподілом** щонайменше за п'ятьма прогонами;
- тест-план називає **обрані сценарії, свідомо непокрите і ціну покриття**;
- останній рядок відповідає на питання «що обміняли · чого не робимо · за яких умов рішення зміниться».

## Карта розділів

| Розділ | Лабораторна | ДЗ_4 |
| --- | --- | --- |
| 1. Контекст і прогноз | прогноз перед стартом | крок 1 |
| 2. Профіль деградації | кроки «Прогнати той самий діалог», «Повторити сценарій п'ять разів», «Скоротити й подовжити сценарій» | крок 2 |
| 3. Механізми збою | кроки «Знайти крок втрати й довести причину», «Знайти й класифікувати самосуперечність» | крок 3 |
| 4. Інваріанти й оракули | крок «Написати дві інваріант-перевірки» | крок 4 |
| 5. Тест-план і ціна | — | крок 5 |
| 6. Залишковий ризик і рішення | — | крок 6 |
| Останній рядок | — | крок 6 |

Діалогові кейси (крок 7 ДЗ) ви записуєте у файл `l05_dialogs.jsonl`, тому окремого розділу для них тут немає.

---

## 1. Контекст і прогноз

*Заповнюєте:* **Лабораторна** — рядок «Прогноз» перед першим кроком. **ДЗ**, крок 1 — решту рядків.

```
система             PayPilot
профілі             clean · lesson-05 (D06, D07)
модель і провайдер  anthropic · claude-haiku-5-5 (LLM_MODEL у .env порожній;
                    модель підтверджена трейсами)
основний сценарій   C-07, 7 кроків, клієнт CUS-0004
набір               l05_dialogs.jsonl, 6 кейсів · set_hash c2c9b03175f6
попередні прогони    2026-10-18, старий набір із 5 кейсів · as_of 2026-09-15
```

**Прогноз до наступного прогону:** у `lesson-05` сума `TX-0401` буде пошкоджена згорткою на кроці 3 у сценаріях, де поріг дорівнює 2; у контрольному `CONV-007` (ті самі 7 реплік, поріг 8) згортки не буде, тож згортка не може бути причиною втрати суми. Для суми baseline перевіряється результатом `get_transactions` для `TX-0401`, а не повторенням суми в репліці агента.

Прогноз для розширеного набору зафіксовано у [файлі прогнозу](./reports/l05_dialogs-extended-forecast.md) і закомічено до наступних прогонів.

---

## 2. Профіль деградації

*У темі: «D06: згортка, яка викидає число», «Момент втрати і момент прояву», «Профіль деградації».*

### 2.1. Лист запису: контрольний прогін на `clean`

*Заповнюєте:* **Лабораторна**, крок «Прогнати той самий діалог на двох профілях».

Профіль `clean` · клієнт `CUS-0004` (кнопка «Від імені») · нова розмова.

#### Актуальний контрольний прогін

Session ID: `8862201854ad`. Повні репліки, результати інструментів і текст згортки збережено в [новому листі лабораторної](./l05-1.md#2-контрольний-прогін-clean). Нижче — скорочений запис трьох обов'язкових складових кожного кроку.

| Крок | Репліка агента (скорочено) | Сутності в історії | Retrieval-контекст кроку |
| --- | --- | --- | --- |
| 1 | `TX-0401` — `duplicate_charge`; eligibility пройдена, дедлайн `2026-09-18`. Спір не відкрито. | У запиті є `240.00 EUR` і `TX-0401`. Результати інструментів цього ходу додають дату `2026-07-20`, код `duplicate_charge`, вікно 60 днів. Це відрізняємо від попередньої історії. | `get_account({})` → `CUS-0004`, `ACC-1005`; `get_transactions` для `ACC-1005`, limit 50 → `TX-0401` / `240 EUR` та інші операції; `check_dispute_eligibility(TX-0401, duplicate_charge)` → eligible, 60 днів, `as_of: 2026-09-15`, deadline `2026-09-18`, усі checks pass. Request ID `091f30a74a73417b`. |
| 2 | Так, можна оскаржити `TX-0401` на `240 EUR` як duplicate charge; дедлайн `2026-09-18`; потрібне підтвердження. | У попередньому ході записані всі чотири сутності: `240.00 EUR`, `2026-07-20`, `duplicate_charge`, 60 днів. Нової згортки немає; текст входу окремо не наведено. | Нових інструментів немає, лише `llm.call`; відповідь використовує попередню перевірку. Request ID `ab34300fcf5141c8`. |
| 3 | Денний transfer limit `5,000 EUR`; окремий card-only limit не підтверджено. Агент нагадує `TX-0401`, `240 EUR` і дедлайн `2026-09-18`. | **Прямий доказ із `summary.text`:** згортка замінила 10 повідомлень і зберегла `240.00 EUR`, дату `2026-07-20`, `duplicate_charge`, 60 днів, також дедлайн і дату перевірки. **Втрати на цьому кроці немає.** | `check_limits(CUS-0004)` → tier1, `as_of: 2026-09-15`; день: limit 5000, spent 0, remaining 5000 EUR; місяць: limit 50000, spent 35.5, remaining 49964.5 EUR. `agent.summarize` — подія згортки, не retrieval. Request ID `ba9fc8977d0a4726`. |
| 4 | FX spread Tier 1 — `1.5%`, allowance `500 EUR` на місяць; для точного quote потрібні додаткові дані. | Остання підтверджена згортка зберігає всі чотири сутності спору. Нової згортки немає; окремого знімка входу немає. `500 EUR` — allowance, не сума спору. | `search_knowledge_base`, query `FX spread tier1 EUR USD conversion fee free allowance`, `kb_clean`, top_k 4: `tariffs.md#s2` (1.8028), `fx-guide.md#s1` (1.5765), `fx-operations.md#s3` (1.4699), `fees-detail.md#s2` (1.1471). Повні результати містять ставку, allowance і правила його застосування. Request ID `18ecc56843bb48f9`. |
| 5 | Повернення до `TX-0401`, `240 EUR`, `2026-07-20`, duplicate charge; дедлайн `2026-09-18`; потрібне «yes». | У згортці збережені всі чотири сутності; нової згортки немає. Агент називає суму без повторного отримання транзакцій. Прояву втрати суми немає. | Нового retrieval / інструментів немає, лише `llm.call`; використовуються попередні дані. Request ID `c1ba682c7fc14ab3`. |
| 6 | Сума `240 EUR` відповідає `TX-0401`; для подання duplicate-charge спору потрібне явне підтвердження. Дедлайн незмінний. | Клієнт повторно вводить `240 EUR`; у попередній згортці є сума, дата, код і 60 днів. Повторення суми саме по собі не доводить її збереження від початку. | У наданому фрагменті лише `llm.call`; JSON обірваний наприкінці, повноту дерева не підтверджено. Request ID `6b6b09add7454a77`. |
| 7 | Дедлайн `2026-09-18`; за датою перевірки `2026-09-15` лишається 3 дні. Спір не подано. | У підтвердженій згортці є всі чотири сутності та дедлайн. Нової згортки немає; відповідь не суперечить кроку 2. | Нового retrieval / перевірки eligibility немає, лише `llm.call`. Використано попередній результат із `as_of: 2026-09-15`. Request ID `f655f1e398bb4074`. |

**Висновок контролю:** на кроці 3 згортка зберегла сутності; на кроці 5 прояву втрати немає; дедлайни кроків 2 і 7 узгоджені. У цьому прогоні втрати суми та самосуперечності щодо строку не виявлено.

#### Попередній контрольний прогін (архів)

Нижче збережено попередній запис; його не змішуємо з актуальним прогоном сесії `8862201854ad`.

| Крок | Репліка клієнта (скорочено) | Репліка агента (скорочено) | Сутності в історії | Retrieval-контекст кроку |
| --- | --- | --- | --- | --- |
| 1 | Give me reason code for transaction TX-0401 with amount 240.00 EUR. | The reason codes and their dispute windows (days from the transaction date) are:<br>Reason code — Use when — Window<br>duplicate_charge — The same payment was charged more than once — 60 days<br>fraud_card_not_present — A card payment was made without your authorization (online/phone) — 120 days<br>goods_not_received — You paid for goods that were not delivered — 90 days<br>service_not_rendered — You paid for a service that was not provided — 90 days<br>unauthorized_debit — A debit was made from your account without your authorization — 56 days<br>The code depends on what happened with TX-0401. Tell me what occurred, and I'll confirm the correct code. I can also check the transaction's eligibility under that code. If you'd like, I can look up TX-0401 on your account first, but I'll need the account ID if you have more than one account. | У репліці клієнта: транзакція `TX-0401`, сума `240.00 EUR`; дата відсутня, reason code не обрано, вікно для справи не визначено. У відповіді агента: `TX-0401` і довідкові пари reason code / вікно: `duplicate_charge` / 60 днів, `fraud_card_not_present` / 120 днів, `goods_not_received` / 90 днів, `service_not_rendered` / 90 днів, `unauthorized_debit` / 56 днів (від дати транзакції). Це не вибраний код і не підтверджене вікно для `TX-0401`. Фактичний склад історії на вході моделі не надано. | `tool.search_knowledge_base`.<br><br>`disputes.md#s1` · `1.6583` · Payment dispute rules<br>**Reason codes and chargeback windows**<br>Each reason code has a window in days from the transaction date. A dispute opened after the window<br><br>`disputes-process.md#s4` · `0.7625` · Dispute handling process<br>**Review timelines**<br>The review target is 10 business days for card reason codes and 20 business days for transfer reason codes. Complex<br><br>`disputes-process.md#s6` · `0.7071` · Dispute handling process<br>**Withdrawing a dispute**<br>A customer may withdraw a dispute at any time before resolution. Withdrawal is final; the same transaction can<br><br>`notifications.md#s2` · `0.6124` · Notifications<br>**Dispute updates**<br>A dispute generates a notification on every state change. The notification names the state, never the internal reason for a dec<br><br>Наведено отримані від користувача уривки; закінчення обрізані. Текст пошукового запиту та повний вміст фрагментів не надано. |
| 2 | Can I dispute it? | Whether TX-0401 can be disputed depends on the reason, because each code has its own window and eligibility rules. What happened with this payment?<br><br>**Charged twice:** duplicate_charge<br>**Card payment you didn't authorize:** fraud_card_not_present<br>**Debit you didn't authorize (not a card payment):** unauthorized_debit<br>**Paid for goods that never arrived:** goods_not_received<br>**Paid for a service that wasn't provided:** service_not_rendered<br><br>Once you tell me, I'll check eligibility for TX-0401 under that code before anything is filed. I'll also confirm the transaction details against your account first. | У наданій історії діалогу з кроку 1: `TX-0401`, сума `240.00 EUR`, довідкові пари reason code / вікно: `duplicate_charge` / 60 днів, `fraud_card_not_present` / 120 днів, `goods_not_received` / 90 днів, `service_not_rendered` / 90 днів, `unauthorized_debit` / 56 днів. Дата відсутня, reason code не обрано, вікно для справи не визначено. У відповіді кроку 2 агент посилається на `TX-0401` і повторює п'ять можливих кодів, але не суму чи числові вікна; це не доводить їх втрату з історії. Фактичний склад історії на вході моделі не надано. | Фактичний retrieval-контекст не надано. Відповідь містить умови застосування п'яти reason codes і згадку про перевірку eligibility, але не підтверджує retrieval-документи, результати перевірки чи отримання деталей транзакції з рахунку. |
| 3 | What is my daily card limit? | Your daily transfer limit is **EUR 5,000**, as of 15 September 2026. You have not used any of it today, so the full EUR 5,000 is available.<br><br>For reference, your monthly limit is EUR 50,000, with EUR 35.50 used so far this month and EUR 49,964.50 remaining. | У наданій попередній історії: `TX-0401`, сума спору `240.00 EUR`; дата транзакції відсутня, reason code не обрано, вікно для справи не визначено; довідкові пари код / вікно з кроку 1 збережені в записі діалогу. Нові сутності у відповіді: денний ліміт переказів `5,000 EUR`, використано за день `0 EUR`, доступно `5,000 EUR`; місячний ліміт `50,000 EUR`, використано `35.50 EUR`, залишок `49,964.50 EUR`; дата актуальності ліміту `2026-09-15` (не дата транзакції). Агент відповів про transfer limit замість запитаного card limit. Фактичний склад історії на вході моделі не надано; відсутність суми спору у відповіді не доводить її втрату. | `tool.get_account`, `tool.check_limits` (за наданими даними про крок). Аргументи викликів і повернуті результати не надано; точний вміст контексту інструментів не встановлено. У відповіді наведено ліміти переказів, а не підтверджений денний ліміт картки. |
| 4 | And the FX spread if I pay in USD? | Under the Tier 1 schedule, the FX spread for conversions is **1.5%** of the gross converted amount. The first **EUR 500** of conversions each calendar month is free of spread, and the allowance resets on the first day of each month.<br><br>Two points for a USD payment:<br>- If the whole conversion fits within your remaining monthly allowance, no spread is charged.<br>- If a conversion crosses the allowance boundary, the 1.5% spread applies to the entire conversion, not just the portion above EUR 500.<br><br>I don't have your remaining conversion allowance for this month, so I can't say whether a given payment would fall inside it. If you tell me the USD amount, I can run a full quote showing the mid-market rate, spread, allowance and final amount. | У наданій попередній історії: `TX-0401`, сума спору `240.00 EUR`; дата транзакції відсутня, reason code не обрано, вікно для справи не визначено; довідкові пари код / вікно з кроку 1 та ліміти з кроку 3 є в записі діалогу. Нові сутності з реплік кроку 4: валюта платежу `USD`, тариф `Tier 1`, FX spread `1.5%` від повної суми конвертації, безспредовий обсяг `500 EUR` на календарний місяць, скидання першого числа місяця; за словами агента, при перетині межі spread застосовується до всієї конвертації. Залишок безспредового обсягу та сума USD-платежу невідомі; `500 EUR` — не сума спору. Фактичний склад історії на вході моделі не надано; відсутність згадки про спір у відповіді не доводить втрату його сутностей. | `tool.search_knowledge_base` · `kb_clean` / `top_k 4` · `1.4ms`.<br><br>`tariffs.md#s2` · `1.8028` · Verta tariff schedule<br>**FX spreads by tier**<br>The spread is applied over the mid-market rate only after the free monthly conversion allowance is exhausted.<br><br>&#124; Tie<br><br>`fx-guide.md#s1` · `1.5765` · Currency conversion guide<br>**How a conversion is priced**<br>A conversion quote has these components, in order: the mid-market rate for the currency pair; the tier s<br><br>`fx-operations.md#s3` · `1.4699` · Currency conversion operations<br>**Allowance accounting**<br>The free monthly conversion allowance is counted in EUR equivalent and resets on the first calendar day o<br><br>`fees-detail.md#s2` · `1.1471` · Fee details<br>**Card payment fee**<br>Card payments carry a percentage fee on the EUR equivalent. No flat fee applies.<br><br>Наведено отримані від користувача уривки; обрізані закінчення не доповнено. Текст пошукового запиту та повний вміст фрагментів не надано. Залишок безспредового обсягу не встановлено, FX quote лише запропоновано. |
| 5 | Fine. Back to that charge — what do I do next? | I found TX-0401 on your account (ACC-1005):<br><br>- **Date:** 20 July 2026<br>- **Merchant:** TechMart Online<br>- **Amount:** EUR 240.00 (card, settled)<br><br>I also see **TX-0402**, an identical EUR 240.00 charge to TechMart Online on 14 July 2026. If you were charged twice for one purchase, that may be relevant.<br><br>To choose the right reason code, I need to know what happened. Which describes your situation?<br><br>1. **Duplicate charge:** You were charged twice for the same purchase. This would point to TX-0401 and TX-0402.<br>2. **Unauthorized:** You did not make or recognize this payment. Please also tell me whether your card was lost, stolen, or used by someone else.<br>3. **Goods or service not delivered:** You paid, but the items or service never arrived or were not provided.<br><br>Once you tell me which applies, I'll check whether the dispute is still eligible. Eligibility depends on the transaction date, and the window for some reason codes may be close to closing. If the case involves a possible fraud and you're unsure what to do, I can also escalate to a human agent. | У наданій попередній історії: `TX-0401`, сума `240.00 EUR`, довідкові пари reason code / вікно з кроку 1; reason code не обрано, вікно для справи не визначено. Нові дані за словами агента: рахунок `ACC-1005`; `TX-0401` — дата `2026-07-20`, продавець `TechMart Online`, `240.00 EUR`, карткова операція зі статусом `settled`; `TX-0402` — дата `2026-07-14`, той самий продавець і сума `240.00 EUR`. Дати й операції розрізнено; можливе подвійне списання — гіпотеза, не підтверджений факт і не обраний reason code. Агент знову називає суму, але це не доводить її збереження саме в історії: він стверджує, що знайшов транзакцію на рахунку. Фактичний склад історії на вході моделі не надано. | `tool.get_transactions` (за наданими даними про крок). Аргументи виклику та повернутий результат не надано; точний вміст контексту інструмента не встановлено. У відповіді агент наводить реквізити `TX-0401` і `TX-0402` на `ACC-1005`. Перевірку eligibility й можливу ескалацію лише запропоновано; конкретний строк / дедлайн спору не встановлено. |
| 6 | Amount 240 euro, as I said at the start | Yes, the amount matches. TX-0401 is EUR 240.00, so I have the right transaction.<br><br>The amount doesn't tell me which reason code applies, though. I still need to know what happened with the TechMart Online charge on 20 July:<br><br>1. **Duplicate charge:** You were charged twice for one purchase. Since TX-0402 is also EUR 240.00 to TechMart Online, this is worth checking.<br>2. **Unauthorized:** You did not make or recognize this payment.<br>3. **Goods or service not delivered:** You paid, but the item or service never arrived or was not provided.<br><br>Tell me which one fits, and I'll check eligibility for TX-0401 before anything is filed. | У наданій попередній історії: `TX-0401`, `240.00 EUR`, дата `2026-07-20`, `TechMart Online`, `ACC-1005`; також `TX-0402`, `240.00 EUR`, дата `2026-07-14` і довідкові пари reason code / вікно з кроку 1. Клієнт повторно вводить суму `240 EUR`; агент підтверджує `TX-0401` / `240.00 EUR`, повторює продавця, дату «20 July» та згадує `TX-0402` із тією самою сумою. Reason code досі не обрано, вікно для справи не визначено, подвійне списання не підтверджено. Повторне введення суми клієнтом не дозволяє довести її збереження з початку діалогу. Фактичний склад історії на вході моделі не надано. | Дані про retrieval / виклики інструментів на кроці 6 не надано; це не означає, що викликів не було. Відповідь повторює реквізити з кроку 5; перевірку eligibility лише запропоновано, результат перевірки не наведено. |
| 7 | And I still have time to file, right? | It depends on the reason code, and the windows are different:<br><br>- **Duplicate charge** (60 days): about 3 days left. Counting from 20 July 2026, the window closes around **18 September 2026**.<br>- **Goods or service not delivered** (90 days): open until about 18 October 2026.<br>- **Unauthorized debit** (56 days): **already closed**. The window ended around 14 September 2026.<br>- **Fraud, card not present** (120 days): open until about 17 November 2026.<br><br>These dates are based on today being 15 September 2026, which is the date in your limits data, and they are estimates. The eligibility check gives the official answer.<br><br>If this is a duplicate charge, the window is closing soon, so please tell me that and I'll run the check right away. If it is an unauthorized payment, the standard unauthorized-debit window has passed, so I would need to check the other codes with you, and a human agent may be able to review it further. | У наданій попередній історії: `TX-0401`, сума `240.00 EUR` (повторно введена на кроці 6), дата транзакції `2026-07-20`, `TechMart Online`, `ACC-1005`; reason code не обрано, довідкові вікна — 60 / 90 / 56 / 120 днів. У відповіді агент наводить орієнтовні дедлайни: duplicate charge — `2026-09-18` (нібито 3 дні лишилося), goods / service not delivered — `2026-10-18`, unauthorized debit — `2026-09-14` (нібито закрито), fraud card not present — `2026-11-17`. Агент явно використовує дату актуальності лімітів `2026-09-15` як «сьогодні», хоча вона не підтверджує поточну дату; дата цього запису — `2026-10-18`. Отже, висновки про залишок часу спираються на непідтверджену поточну дату, а не на перевірку eligibility. Фактичний склад історії на вході моделі не надано. | Дані про retrieval / виклики інструментів на кроці 7 не надано. Агент посилається на дату з limits data (крок 3), дату транзакції (крок 5) та довідкові вікна, наведені на кроці 1; це не підтверджує новий retrieval чи точний склад входу. Перевірку eligibility лише запропоновано; офіційного результату, обраного reason code та підтвердженої поточної дати немає. |

*«Сутності в історії» — що з чотирьох параметрів справи є в історії, яку отримує модель на цьому кроці: сума · дата · reason code · вікно. Якщо історію згорнуто, напишіть «згортка» і що в ній лишилося.*

### 2.2. Лист запису: прогін на `lesson-05`

*Заповнюєте:* **Лабораторна**, той самий крок. Це прогін 1 для розподілу в 2.3.

Профіль `lesson-05` · клієнт `CUS-0004`.

#### Актуальний прогін

Session ID: `6c3f6614123f`. Нову сесію підтверджено ходом `1`; профіль `lesson-05`, дефекти `D06`, `D07`. Повні репліки та результати — у [новому листі](./l05-1.md#актуальна-нова-сесія-6c3f6614123f).

| Крок | Репліка агента (скорочено) | Сутності в історії | Retrieval-контекст кроку | Видно у відповіді, що щось утрачено? |
| --- | --- | --- | --- | --- |
| 1 | `TX-0401`, `240 EUR`, TechMart Online, `2026-07-20`, settled. Пропонує `duplicate_charge` через схожу `TX-0402`; eligibility лише запропоновано. | У запиті `TX-0401`, `240.00 EUR`; результат інструмента додає дату. Код `duplicate_charge` запропоновано у відповіді, не підтверджено клієнтом; вікно ще не визначено. Згортки немає. | `get_account({})` → `CUS-0004`, `ACC-1005`; `get_transactions({"account_id":"ACC-1005","limit":50})` → три операції, включно з `TX-0401` / `240 EUR` / `2026-07-20` і `TX-0402` / `240 EUR` / `2026-07-14`, обидві card, settled. Нових KB / eligibility немає. Request ID `6a99c0c0ff2b4cc0`. | Ні: сума є в запиті, результаті інструмента та відповіді. |
| 2 | `TX-0401` eligible за `duplicate_charge`; сума `240 EUR`, дедлайн `2026-09-18`; потрібне підтвердження подання. | Із кроку 1: `240.00 EUR`, `2026-07-20`, запропонований `duplicate_charge`. Поточна перевірка підтверджує код та вікно 60 днів. Згортки немає. | `check_dispute_eligibility({"transaction_id":"TX-0401","reason_code":"duplicate_charge"})` → eligible true, window_days 60, tx_date `2026-07-20`, as_of `2026-09-15`, deadline `2026-09-18`, reason_code / tx_status / window / compliance_hold — pass. Request ID `55999b82a38c46a9`, session ID `6c3f6614123f`, хід 2. | Ні: агент повторює суму, дату, код і строк; дані узгоджені з інструментом. |
| 3 | Денний transfer limit `5000 EUR`; потім агент називає спір eligible за service-not-rendered, хоча перевіряв `duplicate_charge`. | **Втрата суми на кроці 3 підтверджена `summary.text`.** Після заміни 10 повідомлень немає `240 EUR`, `2026-07-20`, `TX-0401`, 60 днів і дедлайну; код замінено на `service_not_rendered`. Поточна репліка та інструменти не відновлюють суму спору. | `get_account({})` → клієнт і рахунок; `check_limits({"customer_id":"CUS-0004"})` → денний ліміт 5000, використано 0, залишок 5000; місячний 50000, використано 35.5, залишок 49964.5 EUR; as_of `2026-09-15`. Eligibility не викликано. Request ID `b8f0c3bc051b4716`, сесія `6c3f6614123f`, хід 3. Повний текст згортки й результат — у новому листі. | Видно спотворення коду: `duplicate_charge` → `service_not_rendered` і перенесення eligibility. Відсутність згадки суми у відповіді про ліміт ще не визначає прояв її втрати. |
| 4 | FX spread Tier 1 — `1.5%`, allowance `500 EUR`; для quote просить суму конвертації. | Згортка кроку 3 не містить суми спору, дати й вікна; код спотворено. Нової згортки немає. FX-контекст не відновлює реквізити `TX-0401`; `500 EUR` не є сумою спору. | `search_knowledge_base`, query `FX spread EUR to USD tier1 conversion fee free allowance`, `kb_clean`, top_k 4: `tariffs.md#s2` (1.8028), `fx-operations.md#s3` (1.7372), `fx-guide.md#s1` (1.6977), `fees-detail.md#s2` (1.1471). Повні тексти містять потрібні FX-правила. Request ID `f0702ab9e03148f1`, сесія `6c3f6614123f`, хід 4. | Прояву втрати суми спору немає: агент відповідає на іншу тему. Запит суми для FX quote не плутаємо із запитом суми спору. |
| 5 | Просить «yes» для подання за service-not-rendered, стверджує, що eligibility пройдена; операцію називає загально, без суми й ID. | Згортка кроку 3 втратила суму, дату, ID, числове вікно й підмінила код. Поточний запит реквізити не відновлює; нової згортки немає. | Нового retrieval / інструментів немає, лише `llm.call` 5005.2ms; request 5005.8ms. Request ID `af11adcaf9fb4598`, сесія `6c3f6614123f`, хід 5. Не можна назвати відсутній retrieval «цілим»; попередні FX-фрагменти не стосуються спору. | Видно спотворений код і перенесення eligibility з іншого коду. Сума не названа, але повторити її агент не просить; явний прояв втрати саме суми не встановлено. |
| 6 | Знайдено дві операції по `240 EUR`; просить вибрати `TX-0401` або `TX-0402`, потім обіцяє перевірити service-not-rendered. | Сума повторно введена клієнтом; реквізити операцій повернуті інструментом. Згортка кроку 3 не зберегла цільовий ID; новий список операцій не відновлює вибір клієнта. Код досі спотворений, нової згортки немає. | `get_transactions({"account_id":"ACC-1005","limit":20})` → `TX-0403` / `35.5 EUR`, `TX-0401` / `240 EUR` / `2026-07-20`, `TX-0402` / `240 EUR` / `2026-07-14`; усі реквізити цільової операції повернуті. Eligibility не викликано. Request ID `3a2ae7a347844ba1`, сесія `6c3f6614123f`, хід 6. | Видно втрату прив'язки до цільової транзакції: повторне «which one» попри первинну `TX-0401`. Це не тотожне прояву втрати суми, яку клієнт уже повторив. |
| 7 | Обидві операції eligible за service-not-rendered: `TX-0402` до `2026-10-12`, `TX-0401` до `2026-10-18`; знову просить вибрати операцію. | Початковий код підмінено в згортці кроку 3; суму й реквізити повторно отримано на кроці 6. Поточні виклики вже використовують `service_not_rendered`, а не перевірений на кроці 2 `duplicate_charge`. | Дві `check_dispute_eligibility`: для `TX-0401` і `TX-0402` за `service_not_rendered`; обидва результати повні, eligible true, window_days 90, as_of `2026-09-15`, усі checks pass, дедлайни `2026-10-18` і `2026-10-12`. Request ID `738440bbfbfb4e6e`, сесія `6c3f6614123f`, хід 7. | Змінився строк справи: для `TX-0401` було 60 днів / `2026-09-18`, стало 90 / `2026-10-18` через підміну коду в згортці. У термінах лабораторної — D07 поверх D06 (спотворення коду); інструменти правильно перевірили переданий їм код. |

**Підсумок актуальної сесії:** крок втрати суми **3**, доведений текстом згортки; явний крок прояву втрати саме суми не встановлено. Зміна коду видима з кроку 3, зміна строку — на кроці 7. Повні результати eligibility кроку 7 виключають неповний результат цих інструментів як пояснення дедлайну; код був підмінений раніше. Повні докази та порівняння з `clean` — у новому листі.

#### Перервані спроби (архів)

Session ID: `1eeee349f4fc`. Повні репліки й результати інструментів збережено в [новому листі лабораторної](./l05-1.md#3-прогін-lesson-05). Трейс підтверджує активні `D06`, `D07`.

**Перевірка безперервності:** наступний наданий запит «Can I dispute it?» (`13ba724d9b78411c`) має іншу сесію `cadd8c98b815` та номер ходу `1`. Це не крок 2 сесії `1eeee349f4fc`; його окремо збережено в новому листі. Крок 2 нижче залишається незаповненим до отримання продовження тієї самої сесії.

**Додатковий трейс `20b4b16960e64d48`:** початкову репліку з сумою надіслано на ході 3 сесії `cadd8c98b815`, не в новій сесії. `summary.text` цього ходу не містить суми, дати й числового вікна, але згадує `service_not_rendered` та duplicate-charge; поточна репліка повторно вводить `240.00 EUR`. Хід 2 цієї сесії не надано. Доказ збережено окремо в новому листі; він не замінює валідний багатокроковий прогін.

Наступний трейс `974b9d85c4234293` — хід 4 тієї самої додаткової сесії `cadd8c98b815`. На «Can I dispute it?» агент повторює `TX-0401`, `240 EUR`, `duplicate_charge`, дедлайн `2026-09-18`; у трейсі лише `llm.call`, без нових інструментів чи згортки. Збережено окремо, не як крок 2 актуальної таблиці.

| Крок | Репліка агента (скорочено) | Сутності в історії | Retrieval-контекст кроку | Видно у відповіді, що щось утрачено? |
| --- | --- | --- | --- | --- |
| 1 | `TX-0401` — settled card payment `240 EUR`, TechMart Online, `2026-07-20`. Потрібно уточнити причину; `duplicate_charge` можливий, якщо це одна покупка. | У запиті є `TX-0401`, `240.00 EUR`; результат інструмента додає дату `2026-07-20`. Код для справи не обрано, вікно не наведено. Згортки в трейсі немає. | `get_account({})` → `CUS-0004`, `ACC-1005`; `get_transactions({"account_id":"ACC-1005","limit":50})` → `TX-0403` / `35.5 EUR`, `TX-0401` / `240 EUR` / `2026-07-20`, `TX-0402` / `240 EUR` / `2026-07-14`; операції settled, card. KB-пошуку й eligibility немає. Request ID `b02595347b9b4c33`. | Ні: сума й ID є у відповіді та результаті інструмента. Відсутність остаточного коду й вікна не є втратою: їх у цьому прогоні ще не визначено. |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |
| 6 |  |  |  |  |
| 7 |  |  |  |  |

#### Попередній прогін (архів)

Нижче збережено попередній запис; його не змішуємо з актуальною сесією `1eeee349f4fc`.

| Крок | Репліка клієнта (скорочено) | Репліка агента (скорочено) | Сутності в історії | Retrieval-контекст кроку | Видно у відповіді, що щось утрачено? |
| --- | --- | --- | --- | --- | --- |
| 1 | I'm CUS-0004. Give me reason code for transaction TX-0401 with amount 240.00 EUR. | For TX-0401 (EUR 240.00, TechMart Online, 20 July 2026, card, settled), the matching reason code is **duplicate_charge**.<br><br>The account shows a second identical charge: TX-0402, EUR 240.00 to TechMart Online on 14 July 2026, also settled. Two charges of the same amount to the same merchant fit the duplicate_charge reason.<br><br>I have not opened a dispute. If you want to proceed, I'll first check eligibility for TX-0401 under duplicate_charge, then open the dispute only if it is eligible. Before that, please confirm that the two charges were for a single purchase. | У репліці клієнта: `CUS-0004`, `TX-0401`, сума `240.00 EUR`; дата, reason code і вікно не вказані. У відповіді агента: `TX-0401` — `240.00 EUR`, `TechMart Online`, `2026-07-20`, карткова операція, `settled`; `TX-0402` — `240.00 EUR`, той самий продавець, `2026-07-14`, `settled`; агент називає `duplicate_charge` відповідним кодом, хоча клієнт ще не підтвердив, що це одна покупка. Вікно спору не наведено. Фактичний склад історії на вході моделі не надано. | `tool.get_account`, `tool.get_transactions` (за наданими даними про крок). Аргументи викликів і повернуті результати не надано; точний вміст контексту інструментів не встановлено. Перевірку eligibility й відкриття спору лише запропоновано; агент прямо зазначає, що спір не відкрито. | Ознак втрати суми у відповіді немає: агент називає `240.00 EUR`. Це не підтверджує точний склад історії на вході. Окреме спостереження: `duplicate_charge` визначено до підтвердження єдиної покупки; однакові сума й продавець самі по собі не доводять подвійне списання. |
| 2 | Can I dispute it? | I can help with that. Your account ACC-1005 has three recent transactions:<br><br>1. **TX-0403**: 2026-09-01, CityParking, €35.50 (card, settled)<br>2. **TX-0401**: 2026-07-20, TechMart Online, €240.00 (card, settled)<br>3. **TX-0402**: 2026-07-14, TechMart Online, €240.00 (card, settled)<br><br>Which transaction do you want to dispute, and what happened? For example:<br>- **Duplicate charge:** you were charged twice for the same purchase. The two TechMart Online charges of €240.00 may fit this.<br>- **Unauthorized debit:** you did not make or approve the payment.<br>- **Goods not received** or **service not rendered:** you paid but did not receive the item or service.<br>- **Card-not-present fraud:** an online payment was made fraudulently with your card details.<br><br>Once you tell me the transaction and the reason, I'll check whether it is eligible before anything is filed. | У наданій попередній історії: `CUS-0004`, цільова транзакція `TX-0401`, сума `240.00 EUR`, дата `2026-07-20`, `TechMart Online`; також `TX-0402`, `240.00 EUR`, `2026-07-14`; агент назвав `duplicate_charge`, але клієнт не підтвердив причину. У відповіді кроку 2: рахунок `ACC-1005`, обидві TechMart Online транзакції з тими самими сумами й датами; додано `TX-0403`, `2026-09-01`, `CityParking`, `35.50 EUR`; усі три операції — карткові, `settled`. `duplicate_charge` тепер подано як можливий варіант поряд з іншими причинами, а не остаточно визначений код. Вікно спору не наведено. Фактичний склад історії на вході моделі не надано. | `tool.get_account`, `tool.get_transactions` (за наданими даними про крок). Аргументи викликів і повернуті результати не надано; точний вміст контексту інструментів не встановлено. Перевірку eligibility лише запропоновано; результат перевірки не наведено. | Видно повторне уточнення вже названої цільової транзакції: агент питає «Which transaction», хоча клієнт указав `TX-0401` на кроці 1. Це ознака непослідовного використання контексту, але не доказ його втрати чи згортки без входів моделі. Сума `240.00 EUR` є у відповіді, однак могла бути отримана повторно через інструменти. |
| 3 | What is my daily card limit? | Your daily transfer limit on file for your tier (tier1) is **€5,000.00**. As of today (2026-09-15), you have spent €0.00 of it, so **€5,000.00 remains**.<br><br>The limit system reports one daily figure and doesn't separately break out card payments, so I can't confirm whether card spending has its own limit. Your monthly limit is €50,000.00, with €35.50 used and €49,964.50 remaining.<br><br>Do you still want to dispute one of the transactions? If so, tell me which one and the reason. | У наданій попередній історії: `CUS-0004`, `ACC-1005`, цільова транзакція `TX-0401`, сума `240.00 EUR`, дата `2026-07-20`, `TechMart Online`; також `TX-0402` і `TX-0403`. `duplicate_charge` запропоновано, але клієнт не підтвердив причину; вікно спору не наведено. Нові сутності у відповіді: тариф `tier1`, денний ліміт переказів `5,000.00 EUR`, використано `0.00 EUR`, залишок `5,000.00 EUR`; місячний ліміт `50,000.00 EUR`, використано `35.50 EUR`, залишок `49,964.50 EUR`; дату `2026-09-15` агент називає «сьогодні» (дата цього запису — `2026-10-18`; поточну дату середовища прогону не підтверджено). Окремий ліміт картки агент прямо не підтверджує. Фактичний склад історії на вході моделі не надано. | `tool.check_limits` (за наданими даними про крок). Аргументи виклику та повернутий результат не надано; точний вміст контексту інструмента не встановлено. Дані про окремий ліміт картки та підтвердження поточної дати у результаті інструмента не надані. | Агент пам'ятає тему спору, але знову просить назвати транзакцію, хоча `TX-0401` указано на кроці 1. Суму спору у відповіді не повторює; це не доводить її втрату. Без входів моделі не можна підтвердити втрату чи згортку історії. |
| 4 | And the FX spread if I pay in USD? | Your account is Tier 1, and the tariff schedule sets the FX terms for that tier:<br><br>- **FX spread:** 1.5% of the converted amount. This applies to USD conversions like any other currency.<br>- **Free monthly conversion allowance:** EUR 500 (counted in EUR equivalent, resets on the 1st of each month).<br><br>If a conversion fits entirely within your remaining monthly allowance, no spread is charged. Once the allowance is used up, the full 1.5% applies to the whole conversion, including any part that would have fit within the allowance. Your remaining conversion allowance isn't shown in the limit data I can access, so I can't tell you how much of the EUR 500 is still available this month.<br><br>The mid-market rate isn't in my tools, so I can't calculate the exact USD amount. If you tell me how much you want to convert, I can run a full quote showing the mid rate, spread, allowance and final amount. | У наданій попередній історії: `CUS-0004`, `ACC-1005`, `TX-0401`, сума спору `240.00 EUR`, дата `2026-07-20`, `TechMart Online`; `duplicate_charge` запропоновано без підтвердження клієнта, вікно спору не наведено. Нові сутності з реплік кроку 4: валюта `USD`, тариф `Tier 1`, FX spread `1.5%`, безспредовий обсяг `500 EUR` на місяць у EUR-еквіваленті, скидання першого числа; за словами агента, при вичерпанні обсягу spread застосовується до всієї конвертації. Залишок обсягу, сума конвертації та mid-market rate не наведені. `500 EUR` — не сума спору. Фактичний склад історії на вході моделі не надано. | `tool.get_account` · `1.3ms`.<br>`tool.search_knowledge_base` · `kb_clean` / `top_k 4` · `2.3ms`.<br><br>`tariffs.md#s2` · `2.0801` · Verta tariff schedule<br>**FX spreads by tier**<br>The spread is applied over the mid-market rate only after the free monthly conversion allowance is exhausted.<br><br>&#124; Tie<br><br>`fx-guide.md#s1` · `1.819` · Currency conversion guide<br>**How a conversion is priced**<br>A conversion quote has these components, in order: the mid-market rate for the currency pair; the tier s<br><br>`fx-operations.md#s3` · `1.4699` · Currency conversion operations<br>**Allowance accounting**<br>The free monthly conversion allowance is counted in EUR equivalent and resets on the first calendar day o<br><br>`onboarding.md#s2` · `1.4596` · Onboarding and tiers<br>**Tier benefits**<br>A higher tier lowers the FX spread, raises the free monthly conversion allowance and raises both transfer limits.<br><br>Далі в наданому трейсі: `llm.call` · крок `1` · `1839.8ms`; `tool.check_limits` · `1.6ms`. `llm.call` — подія виклику моделі, не retrieval-документ. Аргументи інструментів, результати `get_account` / `check_limits`, текст пошукового запиту та повні KB-фрагменти не надано; обрізані закінчення уривків не доповнено. | Сутності спору у відповіді не згадано через перехід на FX; це саме по собі не доводить їх втрату. Фактичні входи моделі / згортку не надано. Окремо: агент каже, що mid-market rate немає в його інструментах, але водночас пропонує повний quote з цим курсом; можливість такого розрахунку не підтверджено наданим трейсом. |
| 5 | Fine. Back to that charge — what do I do next? | I can't open a dispute yet because I need two things from you: which transaction, and why you want to dispute it. Here are your three most recent transactions:<br>Date — Merchant — Amount — Transaction ID<br>2026-09-01 — CityParking — EUR 35.50 — TX-0403<br>2026-07-20 — TechMart Online — EUR 240.00 — TX-0401<br>2026-07-14 — TechMart Online — EUR 240.00 — TX-0402<br><br>The two TechMart Online charges are identical in amount, so if you were charged twice for one purchase, that could be the issue. Is one of these the charge you mean, and what happened? For example:<br><br>- **Duplicate charge:** you were billed more than once for the same purchase.<br>- **Unauthorized debit:** you did not make or authorize the payment.<br>- **Goods not received:** you paid but did not receive the item.<br>- **Service not rendered:** you paid for a service that was not provided.<br>- **Fraud (card not present):** a payment was made online without your involvement.<br><br>Once you tell me the transaction and the reason, I'll check whether it is eligible under that reason before anything is filed. If you'd rather not work through this yourself, I can also escalate it to a human agent. | У наданій попередній історії: `CUS-0004`, `ACC-1005`, цільова транзакція `TX-0401`, сума `240.00 EUR`, дата `2026-07-20`, `TechMart Online`; також `TX-0402` і `TX-0403`. Клієнт повертається до «that charge», не вводячи новий ID. Агент повторює три операції: `TX-0403` / `2026-09-01` / `CityParking` / `35.50 EUR`; `TX-0401` / `2026-07-20` / `TechMart Online` / `240.00 EUR`; `TX-0402` / `2026-07-14` / `TechMart Online` / `240.00 EUR`. `duplicate_charge` подано як гіпотезу; клієнт причину не підтвердив, вікно спору не наведено. Фактичний склад історії на вході моделі не надано. | `tool.get_transactions` (за наданими даними про крок). Аргументи виклику та повернутий результат не надано; точний вміст контексту інструмента не встановлено. Перевірку eligibility й ескалацію лише запропоновано; результат перевірки або відкриття спору не наведено. | Агент знову просить вибрати транзакцію, хоча `TX-0401` прямо вказано на кроці 1; повернення «that charge» не прив'язано до вже заданої цілі. Це видима непослідовність використання контексту, але не доказ втрати чи згортки без входів моделі. Сума `240.00 EUR` є у відповіді, однак могла бути отримана повторно через `get_transactions`. Уточнення причини обґрунтоване: клієнт її досі не повідомив. |
| 6 | Amount 240 euro, as I said at the start. | Two transactions match EUR 240, both with TechMart Online:<br><br>- **TX-0402**, 2026-07-14, EUR 240.00<br>- **TX-0401**, 2026-07-20, EUR 240.00<br><br>Which one is the charge you mean? And what is the reason: duplicate charge, unauthorized debit, goods not received, service not rendered, or card-not-present fraud? If you were billed twice for one purchase, the duplicate-charge option fits best. Once you confirm both, I'll check eligibility before anything is filed. | У наданій попередній історії: `CUS-0004`, `ACC-1005`, цільова транзакція `TX-0401`, сума `240.00 EUR`, дата `2026-07-20`, `TechMart Online`; також `TX-0402`, `240.00 EUR`, `2026-07-14`. Клієнт повторно вводить `240 EUR`; агент називає обидві транзакції з їхніми датами, продавцем і сумою, але знову просить вибрати цільову. `duplicate_charge` подано умовно — якщо це одна покупка; клієнт причину не підтвердив, вікно спору не наведено. Фактичний склад історії на вході моделі не надано. | Дані про retrieval / виклики інструментів на кроці 6 не надано; це не означає, що викликів не було. Відповідь містить реквізити, уже наведені на попередніх кроках, але їхнє джерело на цьому кроці не встановлено. Перевірку eligibility лише запропоновано; результат перевірки не наведено. | Агент знову питає, яку з двох транзакцій має на увазі клієнт, попри явно задану `TX-0401` на кроці 1. Це видима непослідовність прив'язки до цільової транзакції, але без входів моделі не доводить втрату чи згортку історії. Сума є в поточній репліці клієнта й відповіді агента; її збереження з початку діалогу цим не доведено. Уточнення причини залишається обґрунтованим. |
| 7 | And I still have time to file, right? | Whether you still have time depends on which transaction and which reason. Counting from the transaction dates to today (2026-09-15), per the dispute window rules:<br>Transaction — Date — Days elapsed — Duplicate charge (60 days) — Unauthorized debit (56 days) — Goods not received / Service not rendered (90 days) — Card-not-present fraud (120 days)<br>TX-0401 — 2026-07-20 — 57 — Open, 3 days left — Closed — Open — Open<br>TX-0402 — 2026-07-14 — 63 — Closed — Closed — Open — Open<br><br>The window for the duplicate-charge option on TX-0402 has already closed. TX-0401 is still open under duplicate charge, but only for about 3 more days.<br><br>Tell me which transaction and reason you want, and I'll run the eligibility check before anything is filed. If the window is tight, I'd suggest choosing soon. | У наданій попередній історії: `CUS-0004`, `ACC-1005`, цільова `TX-0401`, сума `240.00 EUR` (повторно введена на кроці 6), дата `2026-07-20`, `TechMart Online`; також `TX-0402`, `240.00 EUR`, `2026-07-14`. Клієнт причину не підтвердив. У відповіді агент наводить вікна: duplicate charge — 60 днів, unauthorized debit — 56, goods not received / service not rendered — 90, card-not-present fraud — 120; обчислює 57 днів для `TX-0401` і 63 для `TX-0402`, називає duplicate-charge вікно першої відкритим із залишком 3 дні, другої — закритим. Розрахунки спираються на «сьогодні» `2026-09-15`, уже назване на кроці 3; поточну дату середовища прогону не підтверджено (дата цього запису — `2026-10-18`). Обраного клієнтом reason code й офіційного результату eligibility немає. Фактичний склад історії на вході моделі не надано. | `tool.search_knowledge_base` · `kb_clean` / `top_k 4` · `2.4ms`.<br><br>`disputes-process.md#s6` · `1.7678` · Dispute handling process<br>**Withdrawing a dispute**<br>A customer may withdraw a dispute at any time before resolution. Withdrawal is final; the same transaction can<br><br>`security-policy.md#s2` · `1.5689` · Customer security policy<br>**Card blocking**<br>A customer may block a card immediately. Blocking is reversible for 30 days; after that a replacement card must be ord<br><br>`disputes.md#s1` · `1.5076` · Payment dispute rules<br>**Reason codes and chargeback windows**<br>Each reason code has a window in days from the transaction date. A dispute opened after the window<br><br>`complaints.md#s1` · `1.4142` · Complaints<br>**Difference from a dispute**<br>A dispute concerns a specific transaction. A complaint concerns the service itself: a delay, an error, or the conduct of<br><br>Наведено отримані від користувача уривки; обрізані закінчення не доповнено. Текст пошукового запиту та повний вміст фрагментів не надано. Релевантний до вікон фрагмент `disputes.md#s1` — третій за оцінкою; наданий уривок не містить числових вікон або підтвердження поточної дати. Перевірку eligibility лише запропоновано. | Агент знову просить вибрати транзакцію попри явно задану `TX-0401` на кроці 1 та розглядає обидві операції. Це видима непослідовність використання цільової транзакції, але не доказ втрати чи згортки без входів моделі. Суму у відповіді не повторено, що не доводить її втрату. Твердження «3 дні лишилося» залежить від непідтвердженої дати «сьогодні»; офіційної перевірки eligibility немає. |

### 2.3. Крок втрати по п'яти прогонах

*Заповнюєте:* **Лабораторна**, крок «Повторити сценарій п'ять разів». **ДЗ**, крок 2 — перевіряєте або доміряєте.

Профіль `lesson-05`, кожен прогін у новій розмові.

Метрика розподілу — **втрата правильного значення суми**: окремо позначаємо видалення та підміну. Перервані спроби виключені; усі п'ять зарахованих прогонів мають по сім кроків.
Первинне джерело лабораторних даних — [журнал прогонів і трейси в `l05-1.md`](./l05-1.md#4-пять-прогонів-lesson-05); наведений нижче розподіл перенесено з його зведення.

| Прогін / сесія | Крок втрати правильного значення суми | Крок прояву пошкодження суми у відповіді |
| --- | --- | --- |
| 1 / `6c3f6614123f` | **3**, видалення; summary request `b8f0c3bc051b4716` | Однозначно не встановлено; повторне уточнення цільового ID на кроці 6 — інший симптом. |
| 2 / `86284c35c1a0` | **3**, видалення; summary request `f09a9d7bdc294544` | Не виявлено за 7 кроків; суму відновлено інструментом на кроці 5. |
| 3 / `4cba37c18f8b` | **3**, підміна 240 → 180 EUR; summary request `36ad6a3699aa4401` | **3**, агент назвав 180 EUR. |
| 4 / `4f1e5eeb0468` | **3**, видалення; summary request `d0ab399c8eeb42e6` | Не виявлено за 7 кроків; уточнення цільового ID на кроці 5 — інший симптом. |
| 5 / `a57a54929e92` | **3**, підміна 240 → 180 EUR; summary request `2ea61fbf09ea45f3` | **5**, request `dee6428fe37c4379`: агент назвав 180 EUR та хибну історію eligibility. |

**Упорядкований ряд кроку втрати правильного значення:** **3 · 3 · 3 · 3 · 3**.
**Діапазон:** від **3** до **3** · **медіана:** **3**.
**Строга метрика «сума відсутня»:** видалення на кроці 3 у **3/5**; у **2/5** сума присутня, але хибна. Повного ряду з п'яти кроків саме видалення немає.
**Спостережувана межа довіри до згорнутої пам'яті:** з **кроку 3** у цій конфігурації; правильна відповідь після tool recovery не доводить цілісності історії. Це не універсальний безпечний поріг для інших сценаріїв.
**Порівняння з прогнозом «крок 3», зафіксованим у журналі:** збіг у 5/5 для втрати правильного значення, але лише 3/5 для буквального зникнення суми.

Повні згортки, відповіді й результати інструментів: [п'ять прогонів у журналі](./l05-1.md#4-пять-прогонів-lesson-05). Крок прояву не підставляємо замість кроку втрати; за невиявленими проявами не обчислюємо штучну медіану.

*Одне число кроку втрати не є результатом: дефект імовірнісний.*

### 2.4. Профіль деградації основного сценарію

*Заповнюєте:* **Лабораторна**, крок «Скоротити й подовжити сценарій» (п. 5). **ДЗ**, крок 2 — уточнюєте за п'ятьма прогонами.

Первинне джерело — [покроковий профіль, входи й трейси в `l05-1.md`](./l05-1.md#профіль-деградації-семикрокової-версії). Нижче — фактичний профіль **прогону №5** з лабораторного журналу, а не усереднена траєкторія. Розподіл кроків втрати та прояву за всіма п'ятьма незалежними сесіями наведено в 2.3; він також перенесений зі зведення в `l05-1.md`.

Один рядок на крок. Момент втрати і момент прояву стоять у різних рядках.

| Крок | Що тримається | Що втрачено | Що було на вході кроку | Механізм | Видно у відповіді |
| --- | --- | --- | --- | --- | --- |
| 1 | TX-0401, 240 EUR, merchant, дата | Не виявлено | Запит, account/transactions | Згортки немає | Ні, збою ще немає |
| 2 | Сума, ID, duplicate_charge, 60 днів, 18 вересня | Не виявлено | Історія й eligibility під duplicate_charge | Згортки немає | Ні, збою ще немає |
| 3 | ID, merchant, settled, спір не відкрито | 240 → 180 EUR; код → service_not_rendered; дата, вікно, дедлайн видалені | summary.text із підміною; limits не містять реквізитів спору | D06 | Ні, приховане пошкодження |
| 4 | FX-тариф, allowance | Реквізити спору не виправлені | Пошкоджена історія, повний FX-retrieval | Наслідок D06 | Ні, приховане пошкодження |
| 5 | ID, merchant, очікування згоди | 180 EUR, хибна історія eligibility під іншим кодом | Пошкоджена історія, нового retrieval немає | Прояв D06; D07 поверх D06 | Так, перший прояв |
| 6 | Відновлено 240 EUR, дати, обидва ID | Історію eligibility не виправлено | Повторена сума, повні account/transactions | Часткове відновлення | Так, хибна історія коду лишається |
| 7 | Сума, ID, новий результат eligibility | Код підмінено; строк 60 → 90 днів | Історія, eligibility під service_not_rendered | Правильний розрахунок для спотвореного параметра | Так, дедлайн 18 жовтня замість 18 вересня |

Покрокову таблицю підтверджено для сесії `a57a54929e92`; вона не замінює розподіл п'яти прогонів у 2.3. Повні докази — [лист запису](./l05-1.md#профіль-деградації-семикрокової-версії).

**Момент втрати / спотворення:** крок **3** · **момент прояву:** крок **5**.
**Кроків із «Видно у відповіді — ні»: 4** (1–4); з них **2 вже після пошкодження історії** (3–4). Кроки 1–2 не є прихованими збоями.
*Такі кроки не потраплять у скаргу, і їх не знайде ручна перевірка. У висновку для команди вони йдуть першими.*

### 2.5. Вісь довжини: скорочення й подовження

*Заповнюєте:* **Лабораторна**, крок «Скоротити й подовжити сценарій» (п. 1–4).

| Версія сценарію | Профіль | Сума зникла? На якому кроці | Що ще втрачено |
| --- | --- | --- | --- |
| 5 кроків | `lesson-05` | **Так, крок 3**; новий get_transactions відновив суму до відповіді. Явного прояву за 5 кроків немає. | ID, merchant, дати, точні вікно й дедлайн видалені зі згортки, потім відновлені інструментами. У згортці змішані service_not_rendered і duplicate-charge; відповіді та перевірки залишилися під duplicate_charge. Сесія e3174cdbbdc2. |
| 7 кроків (основна) | `lesson-05` | Правильне значення втрачено на кроці 3 у 5/5: видалення у 3/5, підміна на 180 EUR у 2/5. | Цільові ID, дати, вікна й дедлайни видалялися; у №1, №3, №5 підмінено код. Пошук іноді відновив факти; деталі в листі п'яти прогонів. |
| 10 кроків (+1 перемикання теми) | `lesson-05` | **Так, крок 3**; get_transactions та eligibility кроку 5 відновили факти. Неправильної суми у відповідях до кроку 10 немає. Валідна сесія `89da0718527e`, step 1–10. | Зі згортки видалено ID, merchant, дати, вікно й дедлайн; змішано service_not_rendered і duplicate-charge. Після recovery поточний код duplicate_charge і дедлайн 2026-09-18 збережені. На кроці 9 проявилося хибне пояснення попередньої історії та приписане клієнту підтвердження причини. Спір не відкритий. |

**Висновок про вісь довжини:** усі три довжини перетнули фактичний момент пошкодження згорткою (крок 3). Коротший діалог не гарантує збереження історії: видимий збій у короткому прогоні приховало інструментальне відновлення. У довгому прогоні додаткові кроки виявили хибну ретроспективну атрибуцію на кроці 9, але додаткової втрати суми, зміни поточного коду чи дедлайну не було. Монотонне погіршення цих цільових сутностей зі збільшенням довжини не підтверджене.

Короткий сценарій вилучає дві побічні теми, довгий додає тему сповіщень, отже порівняння змінює також вісь перемикань, не лише довжину. По одному короткому й довгому запуску та п'ять основних не дають оцінки впливу довжини на частоту збоїв. Відповіді й трейси: [коротка версія](./l05-1.md#коротка-версія--5-кроків-сесія-e3174cdbbdc2), [завершена довга версія](./l05-1.md#довга-версія--завершена-сесія-89da0718527e-10-кроків). Попередні спроби зі зміною session ID залишено окремими архівами, а не об'єднано в один прогін.
*Цей висновок знадобиться в розділі 5, коли обиратимете довжину сценаріїв.*

---

## 3. Механізми збою

*У темі: «Три причини одного симптому», «D06», «D07».*

Правило з теми: перш ніж називати причину, відкрийте вхід кроку. На кроці модель отримує три входи: згорнуту історію, retrieval-контекст кроку і системний промпт.

| Що сталося | Де доказ |
| --- | --- |
| Забув (D06) | сутність була в історії й зникла зі згортки |
| Заперечив себе (D07) | обидва входи цілі, а вихід суперечить сказаному раніше |
| D07 поверх D06 | суперечність, а потрібний параметр зник з історії раніше |
| Не знайшов на цьому кроці | retrieval-контекст кроку порожній або приїхало не те |

### 3.1. Крок втрати і доказ причини

*Заповнюєте:* **Лабораторна**, крок «Знайти крок втрати й довести причину».

```
сесія                             6c3f6614123f, lesson-05
крок втрати                       3 (request b8f0c3bc051b4716)
що було в історії до втрати       TX-0401, 240.00 EUR, 2026-07-20,
                                  duplicate_charge, 60 днів, deadline 2026-09-18
що лишилося в історії після       загальний опис операції, service_not_rendered,
                                  eligibility passed, очікування підтвердження;
                                  суми, ID, дати, числового вікна й дедлайну немає
крок прояву втрати саме суми      однозначно не встановлено;
                                  код спотворено у відповіді кроку 3,
                                  цільову транзакцію повторно уточнено на кроці 6
репліка агента на кроці 6         "Please tell me which one you want disputed."
retrieval очікуваного кроку 5     немає нових викликів, лише llm.call;
                                  оцінка «цілий / неповний» не застосовується
retrieval кроку 6                 get_transactions повернув обидві операції
                                  з повними реквізитами; попередній вибір
                                  клієнта список операцій не відновлює
механізм                          D06: видалення суми та спотворення коду згорткою
```

**Висновок одним реченням:** у сесії `6c3f6614123f` крок втрати суми — **3**, що доведено текстом згортки; явний прояв втрати саме суми на кроці 5 не відтворився, але на кроці 6 видно повторне уточнення видаленого з історії цільового ID попри повний результат `get_transactions`.

Повні докази: [новий лист, «Крок втрати та механізм»](./l05-1.md#крок-втрати-та-механізм). Відсутність нового retrieval на кроці 5 не ототожнюємо з неповним результатом пошуку: результату пошуку там немає. Причину видалення суми встановлено безпосередньо за `summary.text`, а не за текстом відповіді.

#### Класифікація збоїв у п'яти прогонах

Первинні докази входів і трейсів — [журнал `l05-1.md`](./l05-1.md#4-пять-прогонів-lesson-05). **D06** нижче означає, що згортка видалила або підмінила сутність. «Retrieval відсутній» означає, що на кроці не було нового результату, а не те, що пошук повернув неповні дані.

| Прогін / сесія | Збій і механізм | Доказ у вході кроку втрати | Retrieval на кроці втрати / першого прояву |
| --- | --- | --- | --- |
| 1 / `6c3f6614123f` | Крок 3: сума та інші точні сутності видалено, код підмінено — **D06**. На кроці 3 видно спотворення коду; прояв втрати саме суми не встановлено. Крок 6 повторно уточнено цільовий ID — додатковий симптом втрати прив'язки. | До згортки: `TX-0401`, `240 EUR`, `2026-07-20`, `duplicate_charge`, 60 днів. Після: загальний опис, `service_not_rendered`, eligibility passed; суми, ID, дати й вікна немає (`b8f0c3bc051b4716`). | Крок 3: `get_account` і `check_limits` не містять реквізитів спору. Крок 5 — лише `llm.call`; крок 6 `get_transactions` повернув обидві операції повністю. Збою retrieval не встановлено. |
| 2 / `86284c35c1a0` | Крок 3: згортка видалила суму й точні реквізити — **D06**. Неправильної суми у відповіді не виявлено: крок 5 відновив її через інструмент. | `summary.text` на кроці 3 не зберіг `240 EUR`, ID та інші точні поля; зміст можливого дубліката й очікування згоди лишилися (`f09a9d7bdc294544`). | Крок 3 `check_limits` не містить суми спору. На кроці 5 повний `get_transactions` повернув `TX-0401` / `240 EUR` та повні реквізити; eligibility для `duplicate_charge` також повна. **Не «не знайшов»: retrieval відновив дані.** |
| 3 / `4cba37c18f8b` | Крок 3: згортка підмінила `240 → 180 EUR` і `duplicate_charge → service_not_rendered` — **D06**; хибні значення одразу повторено у відповіді. Подальша суперечність — **D07 поверх D06**, не незалежний D07 на цілих входах. | До згортки: правильні `240 EUR` та eligibility для `duplicate_charge`. `summary.text` містить `180 EUR`, інший код і неправдиво приписану йому перевірку (`36ad6a3699aa4401`). | Крок 3 `check_limits` не містить реквізитів спору. Крок 4 `quote_fx` отримав уже хибні `180 EUR` як аргумент; це не retrieval правильного спору. На кроці 7 eligibility-результат повний, але для підміненого коду. Пошукової помилки не встановлено. |
| 4 / `4f1e5eeb0468` | Крок 3: згортка видалила суму й цільовий ID — **D06**. Крок 5 агент повторно питає, яку транзакцію оскаржувати (прояв втрати прив'язки, не окремий доказ втрати суми). Код не підмінено. | До згортки: `TX-0401`, `240 EUR`, `duplicate_charge`; текст згортки зберіг лише загальний зміст duplicate-charge і втратив суму та ID (`d0ab399c8eeb42e6`). | Крок 3 `check_limits` не стосується спору. На кроці 5 повний `get_transactions` повернув обидві операції, але не вибір користувача; сума відновлена до відповіді. Це втрата історичного наміру, не збій retrieval. |
| 5 / `a57a54929e92` | Крок 3: згортка підмінила `240 → 180 EUR` і код — **D06**; перший явний прояв пошкодженої суми у відповіді — крок 5. Зміна дедлайну на кроці 7 — **D07 поверх D06**. | До згортки: `240 EUR`, `duplicate_charge`, 60 днів, дедлайн `2026-09-18`. Після: `180 EUR`, `service_not_rendered`, хибне твердження про eligibility (`2ea61fbf09ea45f3`). | Крок 3 `check_limits` не містить даних спору; крок 5 — лише `llm.call`, нового retrieval немає. На кроці 7 повний eligibility result повернув 90 днів і `2026-10-18` для переданого хибного коду. Пошукової помилки не встановлено. |

**Висновок за трьома механізмами:** у п'яти прогонах доказаний механізм пошкодження — D06 на кроці 3 (видалення у №1, №2, №4; підміна у №3, №5). У №3 і №5 пізніші суперечливі твердження класифікуються як D07 поверх уже пошкодженої згортки; самостійний D07 за цілих входів не доведено. Збою «не знайшов на цьому кроці» не встановлено: ліміти й FX-контекст не призначені для відновлення суми спору, а там, де транзакції або eligibility повторно отримано, результати містили дані для переданих аргументів. Відсутність retrieval на кроці не є доказом його неповноти.

#### Перевірка retrieval на кроці прояву: прогін №5

Для перевірки повноти контексту використовуємо **пізніший прояв зміни дедлайну**, не підміняючи ним перший прояв неправильної суми:

| Подія | Крок і доказ | Контекст інструментів |
| --- | --- | --- |
| Втрата / спотворення історії | **3**, session `a57a54929e92`, request `2ea61fbf09ea45f3`: `summary.text` підмінив 240 → 180 EUR і duplicate_charge → service_not_rendered. | Ліміти не містять реквізитів спору й не відновлюють їх. |
| Перший прояв неправильної суми | **5**, request `dee6428fe37c4379`: 180 EUR і хибне твердження про попередню eligibility під service_not_rendered. | Лише `llm.call`; нового retrieval немає. Статус «цілий / неповний результат» не застосовується. |
| Пізніший прояв: зміна дедлайну | **7**, request `1d885de6b8174acd`: дедлайн 2026-10-18 замість попереднього 2026-09-18. | Повний `check_dispute_eligibility(TX-0401, service_not_rendered)`: `eligible=true`, `reason_code=service_not_rendered`, `window_days=90`, `tx_date=2026-07-20`, `as_of=2026-09-15`, `deadline=2026-10-18`; `reason_code`, `tx_status`, `window`, `compliance_hold` — усі `pass`. |

**Доказ повноти на кроці 7:** результат містить транзакційну дату, переданий код, числове вікно, дату перевірки, дедлайн та всі checks; обрізаного KB-уривка або відсутнього результату тут немає. Розрахунок правильний **для переданого коду**, але сам код спотворений раніше в історії. Отже, це не неповний пошук: причина зміни дедлайну — підміна параметра до виклику інструмента. Це не доказ цілісності історії чи правильності вибору причини спору.

Таким чином, є доказаний крок втрати **3** та прояв із повним інструментальним контекстом **7**. Якщо критерій вимагає повний retrieval саме на **першому прояві суми**, він цими записами не закритий: на кроці 5 виклику немає. Не змінюємо фактичний трейс заради формального виконання. Докладний ланцюжок — у розділі 3.2 та [журналі самосуперечності](./l05-1.md#5-самосуперечність).

### 3.2. Самосуперечність

*Заповнюєте:* **Лабораторна**, крок «Знайти й класифікувати самосуперечність».

```
прогін / сесія                    №5 / a57a54929e92, TX-0401
перша репліка про строк           крок 2: «Dispute window: 60 days from the
                                  transaction date», «Deadline: 2026-09-18»;
                                  duplicate_charge, request 1383657eed8b428f
друга репліка про строк           крок 7: «The dispute window for TX-0401 is
                                  open until 18 October 2026»;
                                  service_not_rendered, tool window_days=90,
                                  request 1d885de6b8174acd
reason code в історії кроку 7     є, але вже підмінений: service_not_rendered;
                                  початковий duplicate_charge спотворено
                                  згорткою кроку 3, request 2ea61fbf09ea45f3
retrieval-контекст кроку 7        повний результат check_dispute_eligibility,
                                  не KB-пошук: 90 днів, deadline 2026-10-18,
                                  as_of 2026-09-15, усі checks pass
класифікація                      D07 поверх D06 (термін лабораторної)
на підставі чого                  клієнт не змінював обставин або причини;
                                  підміна коду прямо зафіксована в summary.text
                                  до пізнішої відповіді та у tool.arguments
```

**Висновок:** дедлайн тієї самої справи змінився з **18 вересня на 18 жовтня** після підміни `duplicate_charge` на `service_not_rendered` згорткою кроку 3. На кроках 5–6 агент також неправдиво приписував попередню eligibility-перевірку новому коду; реально вперше перевірив його на кроці 7.

**Застереження:** код у пізнішій історії не відсутній, а спотворений; тому формальний критерій «код є» не означає цілих входів. Інструмент правильно розрахував 90 днів для переданого коду; збій — необґрунтована зміна підстави спору, не суперечливість двох правил для різних кодів. Незалежний вплив D07 на непошкодженій історії не доведено. Фактичні значення тут **60 → 90 днів**, не демо `120 → 60`; 33 дні у відповіді кроку 7 — залишок від дати стенда, не довжина вікна.

Детальні докази: [лист п'яти прогонів, розділ 5](./l05-1.md#5-самосуперечність).

### 3.3. Контроль на `clean`

*Заповнюєте:* **Лабораторна**, дані з 2.1. **ДЗ**, крок 3 — переносите у висновок.

Контрольний прогін `clean`, сесія `8862201854ad`, має згортку на кроці 3. У `summary.text` збережено цільовий ID `TX-0401` і всі чотири сутності спору: `240.00 EUR`, `2026-07-20`, `duplicate_charge`, 60 днів; також дедлайн `2026-09-18` і `as_of: 2026-09-15`. На кроках 5–7 агент продовжив називати `TX-0401`, суму, дату, код і дедлайн. На кроці 7 дедлайн залишився **`2026-09-18`**; повторної eligibility-перевірки на цьому кроці не було.

**Висновок:** у межах цього контрольного прогону сутності тримаються щонайменше до кроку 7, включно зі згорткою на кроці 3; зміни reason code чи строку не виявлено. Це підтверджує збереження вмісту у записаному тексті згортки й узгодженість відповідей, але не є повним знімком історії на вході моделі кожного кроку. Порівняно з `lesson-05`, де згортки на кроці 3 видалили або спотворили поля, у контрольному `clean` такої втрати не зафіксовано. Джерело й межі доказів: [лабораторний контрольний прогін у `l05-1.md`](./l05-1.md#2-контрольний-прогін-clean).

### 3.4. Збої на інших сценаріях плану

*Заповнюєте:* **ДЗ**, крок 3, після прогонів кейсів із розділу 5.

Історичні прогони набору `594c7901d860` (п'ять кейсів, до виправлення оракулів) виконано 2026-10-18: `clean` — 16/25 повторів, `lesson-05` — 9/25; жоден профіль не мав кейсу, що пройшов усі п'ять повторів. Ці результати не є результатами розширеного набору `c2c9b03175f6`. Нижче класифіковано механізми, підтверджені входами згортки й retrieval, а не самою відповіддю.

| Сценарій | Збій (симптом) | Механізм | Крок втрати | Крок прояву | Доказ: історія · retrieval-контекст |
| --- | --- | --- | --- | --- | --- |
| `CONV-002` · `lesson-05`, run 1 | Сума `TX-0401` змінена з 240 на 180 EUR; цей варіант пройшов на ході 5 у чотирьох із п'яти повторів. | D06 — згортка спотворила сутність. | 3 | 5 | Вхідний `get_transactions` на кроці 1 повернув `TX-0401 = 240 EUR`, і відповідь кроку 1 назвала 240. `summary.text` на кроці 3 записав 180 EUR; на кроці 5 агент повторив 180. [Крок 1](reports/traces/20261018T202705/CONV-002-run1-turn1.json), [згортка кроку 3](reports/traces/20261018T202705/CONV-002-run1-turn3.json), [прояв кроку 5](reports/traces/20261018T202705/CONV-002-run1-turn5.json). |
| `CONV-003` · `lesson-05`, run 1 | Згортка на кроці 3 записала непідтверджений `service_not_rendered`; інструментальний виклик eligibility пішов із цим кодом. | Вигаданий стан у summary; це не D07-суперечність із вибраним клієнтом кодом, бо такого baseline-коду не було. | 3 | 3 | Користувач не назвав причини. `summary.text` кроку 3 приписує клієнтові `service_not_rendered`, а повний `check_dispute_eligibility` викликано з цим кодом; результат `eligible=true`, вікно 90 днів і дедлайн `2026-10-18` правильні саме для переданого коду. [Згортка й інструментальний контекст кроку 3](reports/traces/20261018T202705/CONV-003-run1-turn3.json). |
| `CONV-004` · `lesson-05`, run 1 | Згортка пропустила суму; тест також фіксував відсутність суми у відповіді на запит про дедлайн. | D06 — сума відсутня у `summary.text`; саму відповідь на кроці 10 окремо не вважаємо доказом втрати. | 3 | Не встановлено | `get_transactions` на кроці 1 повернув `TX-0401 = 240 EUR`, але `summary.text` кроку 3 не містить ані суми, ані коду. На кроці 5 агент повторно викликав `get_transactions` і відновив 240 EUR. Отже, втрата у згортці доведена, але тестовий провал на кроці 10 не доводить, що модель не мала цього значення на вході. [Крок 1](reports/traces/20261018T202705/CONV-004-run1-turn1.json), [згортка кроку 3](reports/traces/20261018T202705/CONV-004-run1-turn3.json), [повторне отримання на кроці 5](reports/traces/20261018T202705/CONV-004-run1-turn5.json). |
| `CONV-005` · `lesson-05`, run 2 | Код `duplicate_charge`, підтверджений клієнтом і перевірений на кроці 2, згортка замінила на `service_not_rendered`. | D07 — самосуперечність після коректного retrieval. | 3 | 3 | Репліка клієнта на кроці 2 підтверджує подвійне списання; `check_dispute_eligibility` повернув `duplicate_charge`, 60 днів і дедлайн `2026-09-18`. На кроці 3 `summary.text` уже стверджує, що клієнт підтвердив `service_not_rendered`, а інструмент викликано з ним; результат для іншого коду повний і внутрішньо правильний. [Інструментальний результат кроку 2](reports/traces/20261018T202705/CONV-005-run2-turn2.json) і [згортка та retrieval кроку 3](reports/traces/20261018T202705/CONV-005-run2-turn3.json). |
| `CONV-006` · `lesson-05`, run 1 | Сума `TX-0401` змінена з 240 на 180 EUR; відповідь на кроці 5 не знаходить у транзакціях вигадану суму. | D06 — згортка спотворила сутність. | 3 | 5 | `get_transactions` кроку 1 повернув `TX-0401 = 240 EUR`; `summary.text` кроку 3 записав EUR 180. На кроці 5 агент шукав EUR 180 і не знайшов відповідної операції в поточному retrieval. [Крок 1](reports/traces/20261018T202705/CONV-006-run1-turn1.json), [згортка кроку 3](reports/traces/20261018T202705/CONV-006-run1-turn3.json), [прояв кроку 5](reports/traces/20261018T202705/CONV-006-run1-turn5.json). |

**Що виявили старі прогони `clean`:** 16/25 повторів пройшли, але 0/5 кейсів — усі п'ять повторів. Частина червоних результатів була дефектом тесту, а не доказом дефекту агента:

- У п'яти повторах baseline-сума не повторювалася в репліці на кроці 1. Це не слід вважати відсутнім baseline: передумова має звіряти `get_transactions.result.transactions[id=TX-0401].amount` з `240 EUR`, незалежно від відповіді агента.
- У `CONV-003` екстрактор вибрав `unauthorized_debit` із переліку можливих кодів на кроці 5. Клієнт не назвав причини, тому точний reason code не є інваріантом цього сценарію; кодову перевірку вилучено.
- У `CONV-004` на кроці 9 два відповіді правильно кажуть «reason code not yet chosen», бо клієнт не підтвердив обставини ([run 2](reports/traces/20261018T201040/CONV-004-run2-turn9.json), [run 5](reports/traces/20261018T201040/CONV-004-run5-turn9.json)). На кроці 10 запитано лише строк і статус; три відповіді не повторили суму (наприклад, [run 2](reports/traces/20261018T201040/CONV-004-run2-turn10.json) і [run 5](reports/traces/20261018T201040/CONV-004-run5-turn10.json)), що не є втратою. Тепер сума обов'язкова на кроці 9, необов'язкова на 10, а код не перевіряється як вибраний; окремий L4 ловить лише хибне приписування причини клієнтові.

Отже, `0/5` кейсів на старому наборі не відрізняє шум оракула від дефекту моделі. Усі наступні метрики треба рахувати на новому хеші після повторного прогону; докази D06/D07 із `lesson-05` у таблиці вище лишаються локальними фактами конкретних трейсів, а не статистичним висновком про чистий профіль.

---

## 4. Інваріанти й оракули

*У темі: «Інваріанти: еталон, який агент задає сам собі», «Інваріант у схемі кейса».*

Інваріант перевіряє, що значення не змінилося між кроками. Правильність значення перевіряє оракул на кроці, де воно прозвучало вперше. Кожен кейс має обидва.

### 4.1. Дві перевірки для основного сценарію

*Заповнюєте:* **Лабораторна**, крок «Написати дві інваріант-перевірки».

| Перевірка | Сутність | З кроку | Рівень драбини | `clean` | `lesson-05`: упала в ... прогонах із 5 |
| --- | --- | --- | --- | --- | --- |
| 1 | сума спору `TX-0401`: 240 EUR, допуск 0.01 EUR | 1 | 2 — число з допуском | Старий оракул вимагав повторення суми в відповіді; 3/5. Новий baseline зчитує суму з рядка `get_transactions` за ID, не з відповіді. | **4 з 5**: у трьох сума стала 180 EUR на кроці 5; в одному — на кроці 6 |
| 2 | reason code `TX-0401`: `duplicate_charge` | 2 | 1 — точна рівність нормалізованого коду | 3/5 повторів пройшли; два інші невалідні через відсутню baseline-суму | **0 з 5**: код у всіх п'яти повторах пройшов probes |

**Перевірка 1:** передумова — у трейс-контексті кроку 1 інструмент `get_transactions` повернув рівно один запис з `id=TX-0401`, і його `amount` дорівнює 240 з допуском 0.01. Відповідь агента на кроці 1 для встановлення baseline не аналізується. Далі перевірити названу суму на кроках 2, 5, 6, 7: на кроці 5 значення обов'язкове, на решті відсутність згадки пропускається, але будь-яке назване число порівнюється. Виправлення на кроці 6 не скасовує провалу кроку 5.

**Перевірка 2:** код з кроку 2 порівняти з кодом у відповідях кроків 5, 6, 7. Після нормалізації регістру та роздільників потрібна точна рівність `duplicate_charge`. На кроці 5 код обов'язковий; на інших відсутність згадки не означає зміни. Це перевірка лише умовного кандидата: клієнт ще не підтвердив, що покупки дублювалися.

**Що показали старі перевірки:** на `clean` три повтори пройшли обидва інваріанти, але старий baseline вимагав від агента повторити суму у відповіді; нова передумова цього не вимагає. На `lesson-05` сума порушилась у чотирьох повторах, але код пройшов у всіх п'яти. Це історичні дані хешу `594c7901d860`, не прогнозований результат нового набору.

Старі результати взято з хешу `594c7901d860`; вони використовували текст відповіді як baseline і не перераховуються заднім числом. Новий хеш `c2c9b03175f6` використовує nested tool-result oracle; повторний запуск очікує на виконання. На `lesson-05` підміна 240 → 180 EUR у старому наборі доведена окремо за текстом згортки й трейсами. Історичний звіт і трейси збережені в [README](README.md#прогони) та каталозі [`reports/`](reports).

Виконуваний кейс: [l05_dialogs.jsonl](l05_dialogs.jsonl), `CONV-002`, 7 реплік, `runs: 5`, `min_turns: 7`. У другому інваріанті використано **reason_code**, не `filing_window_days` зі схематичного прикладу завдання. Обмеження extractor-а та запуск описані в [README](README.md#дві-інваріант-перевірки-conv-002).

### 4.2. Інваріанти й оракули всіх кейсів

*Заповнюєте:* **ДЗ**, крок 4. По рядку на кожен кейс із [набору `l05_dialogs.jsonl`](l05_dialogs.jsonl).

Розширений набір кроку 7 містить **шість кейсів**. Старі звіти мають п'ять кейсів і хеш `594c7901d860`; вони не оцінюють виправлені оракули. Для кожного поточного кейсу наведено інваріант, рівень драбини й окремий оракул baseline.

| Кейс | Інваріант: сутність · з кроку · рівень | Оракул: крок · хто перевіряє правильність | Профільний результат |
| --- | --- | --- | --- |
| `CONV-002` · основний; контроль на `clean` | **Сума:** `TX-0401` = 240 EUR, baseline — результат інструмента на кроці 1; проби 2, 5, 6, 7, крок 5 обов'язковий.<br>**Reason code:** `duplicate_charge`, з кроку 2; проби 5, 6, 7, крок 5 обов'язковий. | Сума — `get_transactions.result.transactions[id=TX-0401].amount`, точний рядок транзакції; не відповідь агента. Reason code лишається умовним до клієнтського підтвердження; engine перевіряє eligibility, не правдивість обставин. | Новий хеш `c2c9b03175f6`: повторний прогін очікується. Старий `594c7901d860`: `clean` 3/5, `lesson-05` 1/5; baseline витягувався з відповіді агента. |
| `CONV-003` · короткий, ізоляція довжини | **Сума:** `TX-0401` = 240 EUR, baseline — рядок `get_transactions` за ID; проби 2–5, крок 3 обов'язковий. Reason code тут не перевіряється: клієнт не назвав причину. | Сума — результат `get_transactions` на кроці 1, фільтр `id=TX-0401`, поле `amount`. | Новий хеш `c2c9b03175f6`: повторний прогін очікується. В історичному результаті code matcher помилково вибирав `unauthorized_debit` зі списку кандидатів на кроці 5. |
| `CONV-004` · довгий, найгірший випадок | **Сума:** `TX-0401` = 240 EUR, baseline — рядок `get_transactions` за ID; проби 2, 5, 6, 7, 9, 10; обов'язкові 5 і 9, необов'язковий 10. Reason code не є інваріантом: клієнт його не підтвердив. L4 перевіряє, що агент не приписує клієнту непідтверджену причину на кроці 9. | Сума — результат `get_transactions` на кроці 1, фільтр `id=TX-0401`, поле `amount`. L4 — відповідь на кроці 9 проти реплік клієнта; дати на кроці 10 не є вимогою повторити суму чи код. | Новий хеш `c2c9b03175f6`: повторний прогін очікується. Старий `594c7901d860`: `clean` 2/5, `lesson-05` 1/5; clean провали на кроках 9–10 включали надмірно суворі оракули. |
| `CONV-005` · 7 ходів без побічних тем, ізоляція осі | **Сума:** `TX-0401` = 240 EUR, baseline — рядок `get_transactions` за ID; проби 2, 3, 5–7, крок 5 обов'язковий.<br>**Reason code:** `duplicate_charge`, клієнт підтверджує подвійне списання на кроці 2; проби 3, 5–7, крок 5 обов'язковий. | Сума — результат `get_transactions` на кроці 1, фільтр `id=TX-0401`, поле `amount`. Reason code — клієнтське твердження плюс окрема перевірка eligibility. | Новий хеш `c2c9b03175f6`: повторний прогін очікується. Старий `594c7901d860`: `clean` 4/5, `lesson-05` 4/5. |
| `CONV-006` · одна сутність, ізоляція осі | **Сума:** `TX-0401` = 240 EUR, baseline — рядок `get_transactions` за ID; проби 2, 5–7, кроки 5 і 7 обов'язкові. | Результат `get_transactions` на кроці 1, фільтр `id=TX-0401`, поле `amount`. | Новий хеш `c2c9b03175f6`: повторний прогін очікується. Старий `594c7901d860`: `clean` 4/5, `lesson-05` 2/5. |
| `CONV-007` · контроль без згортки | Ті самі сума та reason code і ті самі 7 реплік, що в `CONV-002`; поріг 8 > довжини 7. Передумова: `agent.summarize` відсутній. | Сума — tool result `get_transactions` на кроці 1; поріг — відсутність span `agent.summarize` до кінця діалогу. Reason code лишається умовним до підтвердження. | Ще не запущено; прогноз у [файлі до прогону](./reports/l05_dialogs-extended-forecast.md). |

**Узгоджена помилка.** Інваріант перевіряє незмінність значення, а не його початкову правдивість. Baseline суми автоматично звіряється з конкретним рядком `get_transactions`, а не з відповіддю, яка може лише повторити помилковий ввід користувача. Для `duplicate_charge` потрібне підтвердження клієнта, що обидва списання стосуються однієї покупки; eligibility engine перевіряє правила й строки, але не цю фактичну обставину. У сценаріях без підтвердження код не є інваріантом.

---

## 5. Тест-план і ціна

*Заповнюєте:* **ДЗ**, крок 5. *У темі: «Вікно росте саме», «Тест-план: три осі покриття і його ціна». Дані з лабораторної: розділ 2.5.*

### 5.1. Осі й повна комбінаторика

Рівні нижче — проєктна сітка для C-07, не твердження, що кожна комбінація вже є окремим кейсом у JSONL.

| Вісь | Значення | Що ловить |
| --- | --- | --- |
| Довжина діалогу | 5 · 7 · 10 ходів клієнта | Раннє спрацювання згортки та додатковий дрейф у довшій розмові. У лабораторії фактично є виміри всіх трьох довжин. |
| Сутності справи | 1 · 2 · 4: сума; сума + reason code; сума + дата + reason code + вікно | Чи впливає кількість точних полів, які треба втримати; усі значення мають бути прив'язані до `TX-0401`. |
| Перемикання теми | 0 · 2 · 3 відгалуження від справи: без відволікань; як у C-07 (ліміти та FX); C-07 плюс одне відгалуження (сповіщення) | Рахуємо вставні побічні теми, а не окремо перехід до теми й повернення. Перевіряємо, чи втрачаються сутності після відволікання та повернення до справи. |

**Повна комбінаторика:** 3 довжини × 3 рівні сутностей × 3 рівні перемикань = **27 сценаріїв**.

**Ціна повної комбінаторики:** сума довжин одного повного проходу — (5 + 7 + 10) × 3 × 3 = **198 ходів на профіль**; за 5 незалежних прогонів — 198 × 5 = **990 ходів-викликів на профіль** (7.33 ходи в середньому × 5 × 27). Якщо запускати повну сітку на обох профілях, це 1980 ходів. Це оцінка за ходами клієнта, не точна кількість внутрішніх LLM- і tool-викликів: один хід агента може містити кілька таких викликів.

### 5.2. Обрані сценарії

Шість JSONL-кейсів планується прогнати по 5 разів на обох профілях. `CONV-002` — основний; `CONV-003` і `CONV-004` — історичні короткий і довгий варіанти, тепер з виправленими оракулами; `CONV-007` — той самий діалог, що й `CONV-002`, із порогом 8, тобто без згортки. Старі звіти охоплюють лише п'ять кейсів і не є виміром нової схеми перевірок.

| Сценарій / стан | Довжина | Сутності | Перемикання | Роль у плані | Яке питання закриває |
| --- | --- | --- | --- | --- | --- |
| `CONV-002` C-07, `lesson-05` — є в наборі, 5 виміряних прогонів | 7 | 4 | 2 | Основний | Базова частота пошкодження сутностей при реальному перемиканні тем. |
| `CONV-003` C-07, короткий — є в наборі; 1 лабораторний вимір | 5 | 4 | 0 | Ізоляція довжини (неповна) | Чи встигають факти загубитися у короткому варіанті; порівняння з основним також змінює перемикання, тому ефект довжини окремо не доводить. |
| `CONV-004` C-07, довгий — є в наборі; 1 лабораторний вимір | 10 | 4 | 3 | Найгірший випадок | Чи проявляється додаткове забруднення історії після теми сповіщень. |
| `CONV-002` C-07, повтор на `clean` — той самий кейс/входи | 7 | 4 | 2 | Контроль | Чи відрізняється збереження тих самих сутностей без активних D06/D07. |
| `CONV-005` C-07, без відгалужень — є в наборі | 7 | 4 | 0 | Ізоляція осі | Порівняно з основним тримає сталі довжину й сутності, прибирає перемикання тем. |
| `CONV-006` C-07, одна відстежувана сутність — є в наборі | 7 | 1 (сума) | 2 | Ізоляція осі | Порівняно з основним тримає сталі довжину й перемикання, зменшує число інваріантів сутностей. |
| `CONV-007` C-07, той самий діалог, поріг згортки 8 | 7 | 4 | 2 | Контроль | Перевіряє той самий набір реплік без `agent.summarize`; відокремлює поведінку без згортки від її пошкодження. |

*Роль: основний / найгірший випадок / контроль / ізоляція осі. Якщо два рядки відповідають на те саме питання, один зайвий.*

### 5.3. Ціна і розклад

**Ціна обраного плану:** шість JSONL-кейсів мають сумарно 7 + 5 + 10 + 7 + 7 + 7 = **43 ходи** за один прохід. П'ять незалежних прогонів коштують 43 × 5 = **215 ходів-викликів на профіль**; запуск набору на `clean` і `lesson-05` коштує **430 ходів-викликів**. Контроль `CONV-007` уже враховано в цих 43 ходах; окремий запуск `CONV-002` на `clean` не додається, бо це той самий case при запуску набору на обох профілях. Це оцінка за ходами діалогу; фактична ціна вища з урахуванням внутрішніх LLM/tool-викликів.

**Порівняння з однокроковим набором `golden.jsonl`:** у файлі **50 кейсів**, отже номінально 50 викликів на один прогін за припущення одного виклику на кейс (250 за 5 прогонів). Діалоговий план коштує 215 ходів на профіль за 5 прогонів; запуск на двох профілях — 430 ходів. Це порівняння номінальних викликів, а не вартості токенів.

**Коли проганяємо діалоговий набір:** перед релізом, п'ять разів у незалежних сесіях на обох профілях; `clean` є контролем для зіставлення з `lesson-05`.

**Чому:** 430 ходів плюс невраховані внутрішні LLM/tool-виклики — помітна ціна для щоденного запуску; п'ять повторів потрібні через нестабільність кроку пошкодження. Передрелізний запуск на оновленому наборі дає перевірку пам'яті, контроль без згортки та базову лінію з оракулом по результату інструмента. `l05_dialogs.jsonl` містить шість кейсів; кожен запускається на `clean` і `lesson-05`.

---

## 6. Залишковий ризик і рішення

*Заповнюєте:* **ДЗ**, крок 6. *У темі: «Чого профіль деградації не ловить», «Три способи виправлення».*

### 6.1. Непокриті комбінації

| Що не покрито | Що станеться, якщо зламається | Чому живемо з цим зараз | Рішення |
| --- | --- | --- | --- |
| Два одночасні спори в одній розмові, кожен зі своєю сумою, датою й reason code (висока кількість сутностей × довга розмова). | Згортка може змішати реквізити різних транзакцій; агент назве суму або строк однієї справи для іншої. | Поточний C-07 містить одну цільову справу; реалізація структурованого стану справи спершу має пройти перевірку на цьому контрольованому випадку. | **Відкласти.** Додати окремий кейс перед запуском кількох активних спорів для клієнтів. |
| Повернення до тієї самої справи після тривалої паузи або зміни мови (10+ кроків × 4 сутності × кілька перемикань). | Часові вікна можуть минути, переклад reason code — змінити значення; відповідь може спиратися на застарілий стан. | Немає в наявному наборі відтворюваного clock-advance та багатомовного трейсу; не можна достовірно оцінити дрейф лише з наявних запусків. | **Відкласти.** Додати кероване просування годинника й мовний варіант до окремого набору ризику. |
| Ізольована варіація кількості сутностей: 7 кроків, 0 тематичних відгалужень, лише сума (порівняно з варіантом із 4 сутностями). | Навіть за відсутності перемикань згортка може спотворити одне числове поле; не буде зрозуміло, чи впливає саме кількість полів. | Є п'ять повторів для базового C-07, але немає такого контрольованого парного сценарію; не варто приписувати причинність наявним порівнянням. | **Прийняти тимчасово.** Ризик обмежений обсягом поточного релізу; додати, якщо зміниться формат згортки або з'явиться багатосутнісна справа. |

### 6.2. Сліпі зони методу

| Сліпа зона | Чим закриваємо або чому приймаємо |
| --- | --- |
| Профіль показує момент втрати, але не першопричину всередині сумаризатора. | Зберігати `summary.text`, склад повідомлень до згортки та її входи/виходи; при зміні алгоритму додати окремий unit-тест сумаризатора на збереження структурованих полів. Сам профіль не приписує причину без цих доказів. |
| Узгоджена помилка: інваріант зелений, якщо помилкове значення стало baseline і повторюється послідовно. | Оракул перевіряє baseline окремо: суму звіряємо з `get_transactions` за `TX-0401`; reason code — з підтвердженими клієнтом обставинами, а eligibility engine перевіряє лише придатність переданого коду. Не вважати зелений інваріант доказом фактичного дублювання покупки. |
| Не тестуємо зміну мови, повернення через добу, дві справи в одній розмові. | Зміна мови й дві справи — відкласти за 6.1 до відповідної функціональності; повернення через добу — прийняти для цього релізу, якщо UI не обіцяє збереження розмови між днями. Переглянути після появи таких сценаріїв або звітів про інциденти. |

### 6.3. Рекомендоване виправлення D06

| Варіант | Обираємо? | Ціна | Що псується натомість |
| --- | --- | --- | --- |
| Винести сутності справи у структуроване поле | **Так** | Зміна схеми стану й сумаризатора; міграція/сумісність старих сесій; окремі перевірки заповнення та прив'язки `amount`, `transaction_id`, `date`, `reason_code`, `window_days` до однієї справи. Для `reason_code` зберігати також джерело/статус (кандидат чи підтверджений клієнтом), а не перетворювати припущення на факт. | Структурований стан може застаріти або розійтися з історією; потрібно визначити джерело істини й оновлювати поле лише з підтверджених tool results / клієнтських фактів. Додається код і супровід схеми. |
| Підняти `SUMMARIZE_AFTER` | Ні | Немає негайної зміни схеми, але зростають витрати контексту й токенів та ризик довшого prompt. | Лише відкладає згортку: довгі діалоги все одно її перетнуть, тому пошкодження не усувається. |
| Дописати «зберігай числа» в промпт сумаризатора | Ні | Низька ціна зміни промпту, але потрібні повторні прогони та контроль регресій. | Інструкція не гарантує точного збереження ID, дат, зв'язків і коду; може збільшити згортку та створити хибне відчуття гарантії. |

**Чому цей варіант під мандатом Ship it:** у п'яти з п'яти прогонів `lesson-05` правильне значення суми пошкоджено вже на згортці кроку 3, тоді як `clean` зберіг усі сутності. Структуровані поля адресують корінь дефекту й можуть бути перевірені до подальшої відповіді; просте відкладання згортки не захищає довгі діалоги, а промпт-інструкція лишається ймовірнісною. Впровадження приймаємо як рішення для виправлення, але перед релізом вимагаємо ті самі п'ять незалежних прогонів і перевірку, що кожне поле прив'язане до `TX-0401`, не підмінене та відновлюється після згортки; reason code має лишатися непідтвердженим кандидатом, доки клієнт не підтвердить обставини.

---

## Останній рядок

```
що обміняли                      додаткову схему й підтримку стану справи на надійніше збереження сутностей крізь згортку
чого не робимо                   не покладаємося лише на довший контекст або інструкцію сумаризатору; не підтримуємо зараз кілька справ і міжмовне повернення
за яких умов рішення зміниться   якщо п'ять прогонів після зміни не збережуть усі поля або стан розійдеться з tool results — переглянемо схему; якщо витрати/міграція неприйнятні — тимчасово піднімемо поріг згортки й повторно оцінимо ризик
```

---

## Чекліст

**Після лабораторної**

- [ ] У розділі 1 записано прогноз
- [ ] Листи запису 2.1 і 2.2 заповнені для всіх семи кроків, з історією і retrieval-контекстом
- [ ] У 2.3 п'ять прогонів, діапазон і медіана
- [ ] У 2.4 момент втрати і момент прояву стоять у різних рядках
- [ ] У 3.1 названо крок втрати і доведено, що retrieval-контекст на кроці прояву цілий
- [ ] У 3.2 самосуперечність класифіковано з доказом
- [ ] У 4.1 дві перевірки з результатом на `clean` і `lesson-05`

**Перед здачею ДЗ_4**

- [ ] У розділі 1 заповнено всі рядки, зокрема кількість кейсів і `set_hash`
- [ ] У 2.3 названо межу довжини діалогу
- [ ] У 3.3 є висновок контрольного прогону, у 3.4 — збої інших сценаріїв або «немає»
- [ ] У 4.2 кожен кейс має інваріант із рівнем драбини і оракул
- [ ] У розділі 5 три осі, щонайменше п'ять сценаріїв з роллю, серед них контроль, ціна повної комбінаторики і плану, розклад
- [ ] У 6.1 щонайменше три непокриті комбінації, хоча б одна «прийняти»
- [ ] У 6.3 обрано виправлення D06 і названо ціну
- [ ] Останній рядок заповнено всіма трьома пунктами
