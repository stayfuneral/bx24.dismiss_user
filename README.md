# Бизнес-процесс по увольнению сотрудника.

Недавно мы столкнулись с проблемой отсутствия передачи дел увольняемыми сотрудниками. Т.е. когда люди уходили, их задачи и клиенты продожали висеть на них, что вносило хаос в работу оставшихся сотрудников.

Поэтому был сконструирован нехитрый бизнес-процесс, который автоматически перекидывает все задачи и базу CRM увольняющегося сотрудника на его непосредственного руководителя, а в назначенный срок блокирует ему доступ на корпоративный портал.

![Шаблон бизнес-процесса](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/00_bp.png)

Итак, давайте заглянем во внутренности процесса и попробуем разобраться, что почём.

### Переменные

![Список переменных](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/02_bp.png)

Процесс содержит несколько переменных:

**`Увольняющийся сотрудник`** *[пользователь]*

**`Руководитель`** *[пользователь]*

**`Дата увольнения`** *[дата]*

**`ID увольняющегося пользователя`** *[строка]*

**`ID руководителя`** *[строка]*

### Выбор увольняющегося сотрудника

После запуска процесса создаётся задание, в котором необходимо выбрать пользователя, а также задать дату увольнения. Оба поля являются обязательными, без них процесс не двинется дальше

![](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/03_bp.png)

![](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/04_bp.png)

### Выбор руководителя

На основе отправленных данных БП получает информацию о руководителе увольняющегося сотрудника, которая записывается в переменную `Руководитель` (действие **изменение переменных**):

![](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/06_bp.png)

![](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/07_bp.png)

### Получение ID выбранных пользователей

Поскольку сырое значение переменных типа Пользователь выглядит `user_XX`, где XX - ID пользователя, то для корректного выполнения PHP-кода необходимо эти айдишники из переменной вытащить. Для этого мы воспользуемся блоком **PHP-код** и с помощью функций `setVariable()` и `str_replace()` присвоим значения переменным **`DismissUserID`** и **`ufHeadID`**:

```php
$this->setVariable("DismissUserID", str_replace('user_', '', $this->getVariable('dismissUser')));
$this->setVariable("ufHeadID", str_replace('user_', '', $this->getVariable('ufHead')));
```

### Пауза в выполнении

Приостанавливает действие процесса до даты увольнения. Также обратите внимание на вставленное значение - мы добавляем не саму переменную, а битриксовую функцию `dateadd()` для того, чтобы прибавить к дате увольнения определённый интервал. Это обусловлено тем, что, как правило, дата увольнения также является и последним рабочим днём сотрудника.

Само же значение выглядит так:

`{{=dateadd({=Variable:dismissDate}, "+18h")}}`

![](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/08_bp.png)

### Передача дел руководителю

С помощью битриксового API ищем все задача и базу CRM увольняющегося сотрудника и вешаем на руководителя. Для удобства работы объявляем новые переменные и присваиваем им значения `DismissUserID` и `ufHeadID`.

```php
$UserDismiss = $this->getVariable('DismissUserID');
$UserSub = $this->getVariable('ufHeadID');

\Bitrix\Main\Loader::includeModule('crm');
\Bitrix\Main\Loader::includeModule('tasks');

$tasksID = [];

$arTaskFilter = [
    '::LOGIC' => 'AND',
    '!REAL_STATUS' => CTasks::STATE_COMPLETED,
    '::SUBFILTER-1' => [
        '::LOGIC' => 'OR',
        '::SUBFILTER-1' => [
            'CREATED_BY' => $UserDismiss
        ],
        '::SUBFILTER-2' => [
            'RESPONSIBLE_ID' => $UserDismiss
        ],
        '::SUBFILTER-3' => [
            'ACCOMPLICE' => [$UserDismiss]
        ],
        '::SUBFILTER-4' => [
            'AUDITOR' => [$UserDismiss]
        ]
    ]
];
$arLeadFilter = [
    'ASSIGNED_BY_ID' => $UserDismiss,
    '!STATUS_ID' => ['CONVERTED', 'JUNK', '1']
];
$arClientsFilter = ['ASSIGNED_BY_ID' => $UserDismiss];
$arLeadSelect = ['ID', 'TITLE', 'ASSIGNED_BY_ID', 'STATUS_ID'];
$arCompanySelect = ['ID', 'TITLE', 'ASSIGNED_BY_ID'];
$arContactSelect = ['ID', 'NAME', 'LAST_NAME', 'ASSIGNED_BY_ID'];

$CrmUpdateFields = ['ASSIGNED_BY_ID' => $UserSub[0]];

$TASK = new CTasks;
$LEAD = new CCrmLead;
$CONTACT = new CCrmContact;
$COMPANY = new CCrmCompany;

$UserAllLeads = CCrmLead::GetListEx([], $arLeadFilter, false, false, $arLeadSelect);
$UserAllContacts = CCrmContact::GetListEx([], $arClientsFilter, false, false, $arContactSelect);
$UserAllCompanies = CCrmCompany::GetListEx([], $arClientsFilter, false, false, $arCompanySelect);
$UserAllTasks = CTasks::GetList([], $arTaskFilter, $arSelect);

while ($Tasks = $UserAllTasks->GetNext(true, false)) {
    $tasksID[] = $Tasks['ID'];
}

while ($leadList = $UserAllLeads->GetNext(true, false)) {
    $LEAD->Update($leadList['ID'], $CrmUpdateFields);
}

while ($contactList = $UserAllContacts->GetNext(true, false)) {
    $CONTACT->Update($contactList['ID'], $CrmUpdateFields);
}

while ($companyList = $UserAllCompanies->GetNext(true, false)) {
    $COMPANY->Update($companyList['ID'], $CrmUpdateFields);
}

if (!empty($tasksID)) {
    
    foreach ($tasksID as $task) {

        $rsTask = CTasks::GetByID($task);
        $arTask = $rsTask->GetNext(true, false);
        if($arTask['CREATED_BY'] == $UserDismiss) {
            $TASK->Update($arTask['ID'],['CREATED_BY' => $UserSub[0]]);
        }

        if ($arTask['RESPONSIBLE_ID'] == $UserDismiss) {
            $TASK->Update($arTask['ID'],['RESPONSIBLE_ID' => $UserSub[0]]);
        }

        if (in_array($UserDismiss, $arTask['ACCOMPLICES'])) {
            $key = array_search($UserDismiss, $arTask['ACCOMPLICES']);
            unset($arTask['ACCOMPLICES'][$key]);
            $arTask['ACCOMPLICES'][] = $UserSub[0];
            $TASK->Update($arTask['ID'], ['ACCOMPLICES' => $arTask['ACCOMPLICES']]);
        }

        if (in_array($UserDismiss, $arTask['AUDITORS'])) {
            $key = array_search($UserDismiss, $arTask['AUDITORS']);
            unset($arTask['AUDITORS'][$key]);
            $arTask['AUDITORS'][] = $UserSub[0];
            $TASK->Update($arTask['ID'], ['AUDITORS' => $arTask['AUDITORS']]);
        }        
    }    
}

```

*Дискуссии о чистоте кода предлагаю не начинать, поскольку тут важнее факт работоспособности, нежели красоты написания.*

### Блокировка пользователя

Опять же, стандартный PHP-код с использованием битриксового API:

```php
$USER->Update($this->getVariable("dismissUser"), ["ACTIVE" => "N"]);
```

*Я целенаправленно не стал пихать все PHP-действия в один блок для удобства пользования*.

### Уведомление

Стандартное уведомление. Получателем может быть кто угодно - админ, руководитель, кадровик и т.п.

![](http://ramapriyakirtan.myjino.ru/bp_151_bx24.dismiss_user/09_bp.png)

На этом всё, приятного пользования ;-)
