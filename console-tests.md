git 8f0031b4ea65784d5786507f76fbf9843c0ea388

---

# Тестирование · Тесты консольных команд

- [Введение](#introduction)
- [Ожидания успеха / неудачи](#success-failure-expectations)
- [Ожидания ввода / вывода](#input-output-expectations)

<a name="introduction"></a>
## Введение

Помимо упрощенного HTTP-тестирования, Laravel предлагает простой API для тестирования [пользовательских консольных команд](artisan) вашего приложения.


<a name="success-failure-expectations"></a>
## Ожидания успеха / неудачи

Для начала давайте рассмотрим, как делать утверждения относительно кода выхода команды Artisan. Для этого мы будем использовать метод `artisan` для вызова Artisan-команды из нашего теста. Затем мы будем использовать метод `assertExitCode`, чтобы подтвердить, что команда завершилась с заданным кодом выхода:

    /**
     * Test a console command.
     *
     * @return void
     */
    public function test_console_command()
    {
        $this->artisan('inspire')->assertExitCode(0);
    }

Вы можете использовать метод `assertNotExitCode` чтобы подтвердить, что команда не завершилась с заданным кодом выхода:

    $this->artisan('inspire')->assertNotExitCode(1);

Конечно, все команды терминала обычно завершаются с кодом состояния `0`, когда они успешны, и с ненулевым кодом выхода, когда они не успешны. Поэтому для удобства вы можете использовать утверждения `assertSuccessful` и `assertFailed` чтобы утверждать, что данная команда завершилась с успешным кодом выхода или нет:

    $this->artisan('inspire')->assertSuccessful();

    $this->artisan('inspire')->assertFailed();

<a name="input-output-expectations"></a>
## Ожидания ввода / вывода

Laravel позволяет вам легко «имитировать» ввод пользователем в консольных командах, используя метод `expectsQuestion`. Кроме того, вы можете указать код выхода / возврата и текст, который вы ожидаете получить от консольной команды, используя методы `assertExitCode` и `expectsOutput`. Например, рассмотрим следующую консольную команду:

    Artisan::command('question', function () {
        $name = $this->ask('What is your name?');

        $language = $this->choice('Which language do you prefer?', [
            'PHP',
            'Ruby',
            'Python',
        ]);

        $this->line('Your name is '.$name.' and you prefer '.$language.'.');
    });

Вы можете протестировать эту команду с помощью следующего теста, который использует методы `expectsQuestion`,` expectsOutput`, `doesntExpectOutput` и `assertExitCode`:

    /**
     * Тестирование консольной команды.
     *
     * @return void
     */
    public function test_console_command()
    {
        $this->artisan('question')
             ->expectsQuestion('What is your name?', 'Taylor Otwell')
             ->expectsQuestion('Which language do you prefer?', 'PHP')
             ->expectsOutput('Your name is Taylor Otwell and you prefer PHP.')
             ->doesntExpectOutput('Your name is Taylor Otwell and you prefer Ruby.')
             ->assertExitCode(0);
    }

<a name="confirmation-expectations"></a>
#### Ожидания подтверждения

При написании команды, которая ожидает подтверждения в виде ответа «да» или «нет», вы можете использовать метод `expectsConfirmation`:

    $this->artisan('module:import')
        ->expectsConfirmation('Do you really wish to run this command?', 'no')
        ->assertExitCode(1);

<a name="table-expectations"></a>
#### Таблица ожиданий

Если ваша команда отображает таблицу информации с использованием метода `table` Artisan, может быть обременительно записывать ожидаемые результаты для всей таблицы. Вместо этого вы можете использовать метод `expectsTable`. Этот метод принимает заголовки таблицы в качестве первого аргумента и данные таблицы в качестве второго аргумента:

    $this->artisan('users:all')
        ->expectsTable([
            'ID',
            'Email',
        ], [
            [1, 'taylor@example.com'],
            [2, 'abigail@example.com'],
        ]);
