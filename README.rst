================
django-robokassa
================

django-robokassa - это приложение для интеграции платежной системы ROBOKASSA в
проекты на Django.

До использования следует ознакомиться с официальной документацией
ROBOKASSA (http://robokassa.ru/Doc/Ru/Interface.aspx). Приложение реализует
протокол взаимодействия, описанный в этом документе.

Установка
=========

::

    $ pip install django-robokassa

Потом следует добавить 'robokassa' в INSTALLED_APPS и выполнить ::

    $ python manage.py syncdb

или, если используется South, ::

    $ python manage.py migrate

Для работы требуется django >= 1.3.1.
Используйте django-robokassa версии 0.9.3, если проект на django 1.2.x или django 1.1.x.

Настройка
=========

В settings.py нужно указать следующие настройки:

* ROBOKASSA_LOGIN - логин
* ROBOKASSA_PASSWORD1 - пароль №1

Необязательные параметры:

* ROBOKASSA_PASSWORD2 - пароль №2. Его можно не указывать, если
  django-robokassa используется только для вывода формы платежа.
  Если django-robokassa используется для приема платежей, то этот
  параметр обязательный.

* ROBOKASSA_USE_POST - используется ли метод POST при приеме результатов от
  ROBOKASSA. По умолчанию - True. Считается, что для Result URL, Success URL и
  Fail URL выбран один и тот же метод.

* ROBOKASSA_STRICT_CHECK - использовать ли строгую проверку (требовать
  предварительного уведомления на ResultURL). По умолчанию - True.

  Механизм строгой проверки:

  1. robokassa.ru, вызывает ResultURL.

  2. Внутри view, связанного с ResultURL, происходит проверка содержащейся в
  запросе md5 подписи через ROBOKASSA_PASSWORD2 - второй пароль, который не
  передается по сети и известен только отправителю и получателю. Он нужен
  для подтверждения того, что запрос был послан именно с robokassa.ru

  3. Через callback, который принимает сигнал из этого view,
  производятся манипуляции внутри сайта (например, начисление средств согласно
  пришедшему запросу).

  4. Затем этот view отправляет на robokassa.ru ответ вида
  HttpResponse("".join(["OK", str(operation_id)]),
  где operation_id - уникальный id текущей операции. Этот ответ необходим в
  том числе для того, чтобы robokassa.ru получила подтверждение того, что все
  необходимые действия произведены.

  5. Если robokassa.ru получает этот ответ, она посылает второй запрос уже на
  SuccessURL, для вывода информативного сообщения пользователю об успешном
  прохождении всей операции. То есть, здесь должен быть только вывод оповещения.

  6. Либо, в случае, если HttpResponse не соответвтует ожидаемому,
  посылается запрос на FailURL, тогда пользователь видит сообщение о 
  произошедшей ошибке.

* ROBOKASSA_TEST_MODE - включен ли тестовый режим. По умолчанию False
  (т.е. включен боевой режим).

* ROBOKASSA_EXTRA_PARAMS - список (list) названий дополнительных параметров,
  которые будут передаваться вместе с запросами. "Shp" к ним приписывать не
  нужно.


Использование
=============

Форма для приема платежей
-------------------------

Для того, чтобы упростить отправку и валидацию данных в
Robokassa, в django-robokassa есть форма RobokassaForm. Она нужна
для вычисления контрольной суммы (в том числе, для проверки того,
что взаимодействие происходит именно с robokassa.ru, а не его подделкой)
и формирования параметров GET-запросов. Для вывода информации в шаблон и
обработки запроса пользователя необходимо реализовать свою собственную форму
для ввода информации о платеже пользователем и view для обработки этой формы.
При POST запросе из этой формы необходимо во view извлекать ее параметры,
создавать form=RobokassaForm() с нужными значениями в initial и делать redirect
на form.get_redirect_url().

Пример view::

    # views.py

    @login_required
    def balance_top_up(request):
        """
        Отправка запроса на пополнение баланса на robokassa.ru
        """

        if request.method == 'POST':

            # Создание экземпляра своей формы после POST запроса
            top_up_balance_form = TopUpBalanceForm(request.POST)

            # Проверка на валидность полей в своей форме
            if top_up_balance_form.is_valid():

                # Здесь необходимо сгенерировать уникальный номер операции op_id

                op_id = ...

                # Генерация контрольной суммы с использованием
                # ROBOKASSA_PASSWORD1 и остальными параметрами.
                # _get_check_sum() см. ниже

                check_sum = _get_check_sum(
                    # Логин пользователя
                    login=settings.ROBOKASSA_LOGIN,
                    # Сумма, на которую происходит повышение баланса
                    rebalancing_sum=\
                        top_up_balance_form.cleaned_data['OutSum'],
                    # Уникальный счетчик операции
                    operation_id=op_id,
                    # Пароль №1
                    password=settings.ROBOKASSA_PASSWORD1
                )

                # Отправка запроса на robokassa.ru для инициации процедуры
                # начисления средств на счет

                form = RobokassaForm(initial={
                    'MrchLogin': settings.ROBOKASSA_LOGIN,
                    'OutSum':
                        top_up_balance_form.cleaned_data['OutSum'],
                    'InvId': op_id,
                    'Desc': 'description',
                    'SignatureValue': check_sum,
                    'Email': request.user.email,
                    'Culture': 'ru'
                })

                # Перенаправление
                return redirect(form.get_redirect_url())
            else:
               # Обработка ошибкок валидации.(Чтобы исключить их наличие
               # после отправки формы, для каждого из ее полей стоит
               # использовать HTML5 атрибут "pattern", содержащий регулярное
               # выражение. Подробнее см. ниже)
    else:
        # Создание экземпляра своей формы для взаимодействия с пользователем.
        # В данном случае форма содержит одно поле - OutSum - сумма перевода.

        form = TopUpBalanceForm()

        # Шаблон см. ниже
        return render(
            request,
            'example.html',
            { 'form': form }
        )

Пример функции подсчета контрольной суммы::

    def _get_check_sum(login, rebalancing_sum, operation_id, password):
        """
        md5 checksum
        """

        data = u':'.join([
            u"%s" % str(login),
            u"%s" % str(rebalancing_sum),
            u"%s" % str(operation_id),
            u"%s" % password
        ])

        return hashlib.md5(data).hexdigest()

Пример формы::

    # forms.py

    # Неотрицательное целое число
    _regexp_template = r'^\d+$'
    _regexp = u'\d+'

    class TopUpBalanceForm(forms.Form):
        """
        Пополнение баланса
        """

        OutSum = forms.CharField(
            max_length=15,
            label=_(u"Введите сумму в рублях для изменения баланса"),
            widget=forms.TextInput(
                attrs={
                    'pattern': _regexp,
                    'maxlength': '7',
                    'placeholder': u'Сумма перевода',
                    'required': 'required',
                    'value': 100500
                }
            )
        )

        def clean(self):
            cleaned_data = super(TopUpBalanceForm, self).clean()
            OutSum = cleaned_data['OutSum']
            if not re.match(_regexp_template, OutSum, re.UNICODE):
                raise forms.ValidationError(_(u'Неверная сумма перевода'))
            return cleaned_data

В initial все параметры необязательны. Детальную справку по параметрам
лучше посмотреть в `документации <http://robokassa.ru/ru/Doc/Ru/Interface.aspx#222>`_
к Robokassa. Можно передавать в initial значения "пользовательских параметров",
описанных в ROBOKASSA_EXTRA_PARAMS ('shp' к ним приписывать опять не нужно).

Соответствующий шаблон::

    {% extends 'base.html' %}

    {% load i18n %}

    {% block title %}{% trans "Пополнение баланса" %}{% endblock %}

    {% block content %}
        <form action="{% url app_name.views.balance_top_up %}" method="post">
            {% csrf_token %}
            {{ form.as_p }}
            <button type="submit">{% trans "Пополнение баланса" %}</button>
        </form>
    {% endblock %}


Получение результатов платежей
------------------------------
В Robokassa есть несколько методов определения результата платежа:

1. При переходе на страницы Success и Fail гарантируется, что платеж
   соответственно прошел и не прошел

2. При успешном или неудачном платеже Robokassa отправляет POST или GET запрос
   на Result URL.

3. Можно запрашивать статус платежа через XML-сервис.

В django-robokassa на данный момент поддерживаются методы 1 и 2 и их совмещение
(дополнительное подтверждение сначала через ResultURL, а затем переход на
SuccessURL при использовании опции ROBOKASSA_STRICT_CHECK = True,
рекомендуется для безопасного обмена данными).
Обработчики подключаются через urls.py, рендерят соответствующие
шаблоны и шлют сигналы в зависимости от успешности платежа.
Также можно сделать свои views, и подключить их самостоятельно через urls.py,
а формы робокассы использовать для валидации данных.


Сигналы
-------
Обработку смены статусов покупок следует осуществлять в обработчиках сигналов.

* robokassa.signals.result_received - шлется при получении уведомления от
  Robokassa. Получение этого сигнала означает, что оплата была успешной.
  В качестве sender передается экземпляр модели SuccessNotification, у
  которой есть атрибуты InvId и OutSum.

* robokassa.signals.success_page_visited - шлется при переходе пользователя
  на страницу успешной оплаты. Этот сигнал следует использовать вместо
  result_received, если не используется строгая проверка
  (ROBOKASSA_STRICT_CHECK=False)

* robokassa.signals.fail_page_visited - шлется при переходе пользователя
  на страницу ошибки оплаты. Получение этого сигнала означает, что оплата
  не была произведена. В обработчике следует осуществлять разблокирвку товара
  на складе и т.д.

Все сигналы получают параметры InvId (номер заказа), OutSum (сумма оплаты) и
extra (словарь с дополнительными параметрами, описанными в
ROBOKASSA_EXTRA_PARAMS).

Пример::

    from robokassa.signals import result_received
    from my_app.models import Order

    def payment_received(sender, **kwargs):
        order = Order.objects.get(id=kwargs['InvId'])
        order.status = 'paid'
        order.paid_sum = kwargs['OutSum']
        order.extra_param = kwargs['extra']['my_param']
        order.save()

    result_received.connect(payment_received)


urls.py
-------

Для настройки Result URL, Success URL и Fail URL можно подключить
модуль robokassa.urls::

    urlpatterns = patterns('',
        #...
        url(r'^robokassa/', include('robokassa.urls')),
        #...
    )

Адреса, которые нужно указывать в панели robokassa, в этом случае будут иметь вид

* Result URL: ``http://yoursite.ru/robokassa/result/``
* Success URL: ``http://yoursite.ru/robokassa/success/``
* Fail URL: ``http://yoursite.ru/robokassa/fail/``


Шаблоны
-------

* ``robokassa/success.html`` - показывается в случае успешной оплаты. В
  контексте есть переменная form типа ``SuccessRedirectForm``, InvId
  и OutSum с параметрами заказа, а также все дополнительные параметры, описанные
  в ROBOKASSA_EXTRA_PARAMS.

* ``robokassa/fail.html`` - показывается в случае неуспешной оплаты. В
  контексте есть переменная form типа ``FailRedirectForm``, InvId
  и OutSum с параметрами заказа, а также все дополнительные параметры, описанные
  в ROBOKASSA_EXTRA_PARAMS.

* ``robokassa/error.html`` - показывается при ошибочном запросе к странице
  "успех" или "неудача" (например, при ошибке в контрольной сумме). В контексте
  есть переменная form класса ``FailRedirectForm`` или ``SuccessRedirectForm``.

Разработка
==========

Разработка ведется на bitbucket и github:

* https://bitbucket.org/kmike/django-robokassa/
* https://github.com/kmike/django-robokassa

Пожелания, идеи, баг-репорты и тд. пишите в трекер: https://bitbucket.org/kmike/django-robokassa/issues

Лицензия - MIT.

Тестирование
------------

Для запуска тестов установите `tox <http://tox.testrun.org/>`_, склонируйте репозиторий
и выполните команду

::

    $ tox

из корня репозитория.
