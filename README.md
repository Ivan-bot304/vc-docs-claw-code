# Модуль commands

## Общее описание

Модуль `commands` — это центральный компонент CLI-агента `claw`, отвечающий за парсинг, валидацию и обработку slash-команд в интерактивном режиме (REPL). Модуль реализует 141 slash-команду (по состоянию на текущую версию), которые позволяют пользователю управлять сессией, работать с рабочей областью, управлять агентами, навыками, MCP-серверами и многим другим.

## Архитектура

Модуль построен вокруг следующих ключевых компонентов:

```
┌─────────────────────────────────────────────────────────────┐
│                     Slash Command Parser                    │
│  validate_slash_command_input() → Result<SlashCommand>      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Command Registry                          │
│  CommandRegistry → entries: [CommandManifestEntry]          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Command Handlers                         │
│  handle_plugins_slash_command()                             │
│  handle_agents_slash_command()                              │
│  handle_mcp_slash_command()                                 │
│  handle_skills_slash_command()                              │
│  handle_slash_command() → main entry point                  │
└─────────────────────────────────────────────────────────────┘
```

## Типы данных

### CommandSource

Перечисление источников команд:

```rust
pub enum CommandSource {
    Builtin,        // Встроенная команда
    InternalOnly,   // Внутренняя команда
    FeatureGated,   // Команда с включением по фиче
}
```

### SlashCommandSpec

Спецификация slash-команды:

```rust
pub struct SlashCommandSpec {
    pub name: &'static str,                // Имя команды
    pub aliases: &'static [&'static str],  // Алиасы команды
    pub summary: &'static str,             // Краткое описание
    pub argument_hint: Option<&'static str>, // Подсказка по аргументам
    pub resume_supported: bool,            // Поддержка resume
}
```

**Пример использования**:

```rust
SlashCommandSpec {
    name: "model",
    aliases: &[],
    summary: "Show or switch the active model",
    argument_hint: Some("[model]"),
    resume_supported: false,
}
```

### SlashCommand

Представление конкретной slash-команды с разобранными аргументами:

```rust
pub enum SlashCommand {
    Help,
    Status,
    Model { model: Option<String> },
    Permissions { mode: Option<String> },
    Plugins { action: Option<String>, target: Option<String> },
    // ... 141 вариантов
    Unknown(String),
}
```

### SlashCommandParseError

Ошибка парсинга slash-команды:

```rust
pub struct SlashCommandParseError {
    message: String,
}
```

**Пример использования**:

```rust
let error = SlashCommand::parse("/unknown extra args").unwrap_err();
assert_eq!(error.to_string(), "Unknown slash command 'unknown'. Use /help to list available slash commands.");
```

### CommandRegistry

Реестр манифестов команд:

```rust
pub struct CommandRegistry {
    entries: Vec<CommandManifestEntry>,
}
```

### CommandManifestEntry

Запись в манифете команды:

```rust
pub struct CommandManifestEntry {
    pub name: String,
    pub source: CommandSource,
}
```

## Публичное API

### Экспортируемые типы

- `CommandRegistry` - реестр манифестов команд
- `CommandSource` - перечисление источников команд
- `SlashCommandSpec` - спецификация slash-команды
- `SlashCommand` - перечисление всех команд
- `SlashCommandParseError` - ошибка парсинга

### Экспортируемые функции

- `validate_slash_command_input()` - парсинг и валидация команд
- `render_slash_command_help()` - генерация полной справки
- `slash_command_specs()` - получение спецификаций
- `resume_supported_slash_commands()` - получение команд с поддержкой resume
- `suggest_slash_commands()` - автодополнение команд
- `render_slash_command_help_detail()` - детальная справка по одной команде
- `handle_slash_command()` - главная точка входа для обработки
- `handle_plugins_slash_command()` - обработка команд плагинов
- `handle_agents_slash_command()` - обработка команд агентов
- `handle_skills_slash_command()` - обработка команд навыков
- `handle_mcp_slash_command()` - обработка MCP-команд

### Стабильность API

Все публичные функции и типы гарантируют обратную совместимость в пределах одной мажорной версии.

## Основные функции

### validate_slash_command_input

**Назначение**: Парсинг и валидация входной строки slash-команды.

**Сигнатура**:
```rust
pub fn validate_slash_command_input(input: &str) 
    -> Result<Option<SlashCommand>, SlashCommandParseError>
```

**Параметры**:
- `input` - строка для парсинга (должна начинаться с `/`)

**Возвращает**:
- `Ok(Some(SlashCommand))` - успешно разобранная команда
- `Ok(None)` - входная строка не является slash-командой
- `Err(SlashCommandParseError)` - ошибка парсинга

**Поведение**:
- Проверяет наличие префикса `/`
- Извлекает имя команды и аргументы
- Сопоставляет с известными спецификациями команд
- Возвращает структурированный результат

**Примеры**:
```rust
validate_slash_command_input("/help")?  // Ok(Some(SlashCommand::Help))
validate_slash_command_input("/model claude-opus")?  
    // Ok(Some(SlashCommand::Model { model: Some("claude-opus".to_string()) }))
validate_slash_command_input("hello")?  // Ok(None) - не slash-команда
```

---

### render_slash_command_help

**Назначение**: Генерация полной справки по всем slash-командам.

**Сигнатура**:
```rust
pub fn render_slash_command_help() -> String
```

**Возвращает**: Строка с форматированной справкой, сгруппированной по категориям.

**Категории**:
- Session & visibility (сессия и видимость)
- Workspace & git (рабочая область и git)
- Discovery & debugging (обнаружение и отладка)
- Analysis & automation (анализ и автоматизация)
- Appearance & input (внешний вид и ввод)
- Communication & control (коммуникация и управление)

---

### slash_command_specs

**Назначение**: Получение списка всех спецификаций команд.

**Сигнатура**:
```rust
pub fn slash_command_specs() -> &'static [SlashCommandSpec]
```

**Возвращает**: Ссылка на статический массив спецификаций команд.

---

### resume_supported_slash_commands

**Назначение**: Получение списка команд, поддерживающих resume.

**Сигнатура**:
```rust
pub fn resume_supported_slash_commands() -> Vec<&'static SlashCommandSpec>
```

**Возвращает**: Вектор ссылок на спецификации команд с `resume_supported = true`.

**Resume-поддержка**:
Поле `resume_supported` указывает, поддерживает ли команда параметр `--resume <session-path>`. Команды с `resume_supported = true` могут работать без активной сессии и используются для загрузки/управления сессиями. Примеры: `/help`, `/status`, `/model`.

---

### suggest_slash_commands

**Назначение**: Автодополнение slash-команд с использованием расстояния Левенштейна.

**Сигнатура**:
```rust
pub fn suggest_slash_commands(input: &str, limit: usize) -> Vec<String>
```

**Параметры**:
- `input` - частичный ввод для поиска совпадений
- `limit` - максимальное количество предложений

**Возвращает**: Вектор строк с предложениями команд.

**Алгоритм**:
1. Вычисляет расстояние Левенштейна между запросом и именами команд
2. Приоритизирует совпадения по префиксу (приоритет 0) и.contains() совпадения (приоритет 1)
3. Отбрасывает результаты с расстоянием > 2 и приоритетом > 1
4. Возвращает отсортированный и ограниченный список

**Пример**:
```rust
let suggestions = suggest_slash_commands("plu", 5);
// ["plugin", "plugins", "plugins install", ...]
```

---

### render_slash_command_help_detail

**Назначение**: Получение детальной справки по одной команде.

**Сигнатура**:
```rust
pub fn render_slash_command_help_detail(name: &str) -> Option<String>
```

**Параметры**:
- `name` - имя команды для получения справки

**Возвращает**: `Some(String)` с детальной информацией или `None`, если команда не найдена.

---

### handle_plugins_slash_command

**Назначение**: Обработка команд управления плагинами.

**Сигнатура**:
```rust
pub fn handle_plugins_slash_command(
    action: Option<&str>,
    target: Option<&str>,
    manager: &mut PluginManager,
) -> Result<PluginsCommandResult, PluginError>
```

**Действия**:
- `None` или `list` - список установленных плагинов
- `Some("install")` - установка плагина по пути
- `Some("enable")` - включение плагина по имени
- `Some("disable")` - отключение плагина по имени
- `Some("uninstall")` - удаление плагина по ID
- `Some("update")` - обновление плагина по ID

**Возвращает**:
- `Ok(PluginsCommandResult)` - результат с сообщением и флагом перезагрузки
- `Err(PluginError)` - ошибка операции с плагинами

**Пример**:
```rust
let mut manager = PluginManager::new();
let result = handle_plugins_slash_command(Some("list"), None, &mut manager)?;
println!("{}", result.message);
```

---

### handle_agents_slash_command

**Назначение**: Обработка команд управления агентами.

**Сигнатура**:
```rust
pub fn handle_agents_slash_command(args: Option<&str>, cwd: &Path) -> std::io::Result<String>
```

**Параметры**:
- `args` - аргументы команды (list, help, путь к справке)
- `cwd` - текущая рабочая директория для поиска определений

**Действия**:
- `None` или `list` - список настроенных агентов
- `Some("help")` - справка по агентам
- `Some(path)` - справка по конкретному агенту

**Возвращает**: Отформатированную строку с результатом или ошибку ввода/вывода.

---

### handle_mcp_slash_command

**Назначение**: Обработка команд MCP (Model Context Protocol).

**Сигнатура**:
```rust
pub fn handle_mcp_slash_command(args: Option<&str>, cwd: &Path) -> Result<String, runtime::ConfigError>
```

**Параметры**:
- `args` - аргументы команды (list, show <server>, help)
- `cwd` - текущая рабочая директория

**Действия**:
- `None` или `list` - список настроенных MCP-серверов
- `Some("show")` - информация о конкретном сервере
- `Some("help")` - справка по MCP

**Возвращает**: Строка с результатом или ошибку конфигурации.

---

### handle_skills_slash_command

**Назначение**: Обработка команд управления навыками.

**Сигнатура**:
```rust
pub fn handle_skills_slash_command(args: Option<&str>, cwd: &Path) -> std::io::Result<String>
```

**Параметры**:
- `args` - аргументы команды (list, install <path>, help)
- `cwd` - текущая рабочая директория

**Действия**:
- `None` или `list` - список доступных навыков
- `Some("install <path>")` - установка навыка по пути
- `Some("help")` - справка по навыкам

**Возвращает**: Отформатированную строку с результатом или ошибку ввода/вывода.

---

### handle_slash_command

**Назначение**: Главная точка входа для обработки всех slash-команд.

**Сигнатура**:
```rust
pub fn handle_slash_command(
    command: &str,
    session: &Session,
    config: CompactionConfig,
) -> Option<HandleCommandResult>
```

**Параметры**:
- `command` - строка slash-команды для обработки
- `session` - ссылка на текущую сессию
- `config` - конфигурация компактации сессии

**Возвращает**:
- `Some(HandleCommandResult)` - если команда была распознана и обработана
- `None` - если команда не распознана или требует runtime-обработки

**Поведение**:
- Парсит входную строку с помощью `validate_slash_command_input`
- Направляет команду в соответствующий специализированный обработчик
- Обрабатывает команды без изменений состояния (например, `/help`)
- Возвращает результат обработки в виде `HandleCommandResult`

**Потенциальные ошибки**:
- `SlashCommandParseError` - если команда имеет неверный синтаксис (генерируется парсером)

---

## Примеры использования

### Базовый парсинг

```rust
use commands::validate_slash_command_input;

match validate_slash_command_input("/plugins list")? {
    Some(SlashCommand::Plugins { action, target }) => {
        println!("Action: {:?}, Target: {:?}", action, target);
    }
    _ => println!("Unknown command"),
}
```

### Обработка ошибок парсинга

```rust
use commands::{validate_slash_command_input, SlashCommandParseError};

fn process_input(input: &str) -> Result<(), SlashCommandParseError> {
    match validate_slash_command_input(input) {
        Ok(Some(command)) => {
            println!("Parsed command: {:?}", command);
            Ok(())
        }
        Ok(None) => {
            println!("Not a slash command");
            Ok(())
        }
        Err(e) => {
            eprintln!("Failed to parse command: {}", e);
            // Предоставить пользователю контекстную помощь
            eprintln!("Available commands: {}", commands::render_slash_command_help());
            Err(e)
        }
    }
}
```

### Обработка результатов команд

```rust
use commands::{handle_slash_command, CompactionConfig};
use runtime::Session;

fn execute_command(session: &mut Session, input: &str) {
    if let Some(result) = handle_slash_command(input, session, CompactionConfig::default()) {
        println!("{}", result.message);
        // Обновить сессию если команда изменила её
        if result.session != *session {
            *session = result.session;
        }
    }
}
```

### Получение справки

```rust
use commands::render_slash_command_help;

let help_text = render_slash_command_help();
println!("{}", help_text);
```

### Автодополнение

```rust
use commands::suggest_slash_commands;

let suggestions = suggest_slash_commands("plu", 5);
// ["plugin", "plugins", "plugins install", ...]
```

## Статистика

- **Количество slash-команд**: 141 (по состоянию на v0.2.0)
- **Количество категорий**: 6
- **Публичные функции**: 11 (см. "Публичное API")
- **Типы данных**: 7 (см. "Публичное API")
- **Строка кода (не документация)**: ~4500

## Зависимости

- `plugins` - модуль управления плагинами
- `runtime` - модуль runtime (конфиги, сессии)
- `serde_json` - сериализация/десериализация JSON

## Интеграция

Модуль `commands` используется в:

1. `rusty-claude-cli` - основной CLI с поддержкой REPL
2. Обработка ввода пользователя в интерактивном режиме
3. Генерация справки и автодополнения
4. Управление плагинами, агентами, навыками, MCP-серверами

## Развитие

При добавлении новой slash-команды необходимо:

1. **Добавить вариант в перечисление `SlashCommand`**
   - Определить структуру аргументов (если есть)
   - Следовать существующим конвенциям именования

2. **Добавить спецификацию в массив `SLASH_COMMAND_SPECS`**
   - Указать имя команды и алиасы
   - Краткое описание (одна строка)
   - Подсказку по аргументам (если есть)
   - Флаг `resume_supported`

3. **Реализовать парсинг в функции `validate_slash_command_input`**
   - Добавить ветку match для имени команды
   - Обработать аргументы с помощью вспомогательных функций:
     - `validate_no_args()` - для команд без аргументов
     - `optional_single_arg()` - для необязательных аргументов
     - `require_remainder()` - для обязательных аргументов
   - Обработать ошибки парсинга

4. **Реализовать обработчик команды**
   - Добавить функцию обработки в соответствующий модуль (plugins, agents, skills, mcp, runtime)
   - Следовать принципу чистоты (если команда не изменяет состояние)
   - Обработать возможные ошибки

5. **Добавить категорию в функцию `slash_command_category`**
   - Выбрать подходящую категорию из: Session, Workspace, Discovery, Analysis, Appearance, Communication
   - Категория определяет группировку в справке

6. **Написать тесты**
   - Добавить тест в `parsing_supported_slash_commands()`
   - Добавить тесты ошибок парсинга
   - Проверить категорию команды

### Resume-поддержка

Поле `resume_supported` указывает, поддерживает ли команда параметр `--resume <session-path>`. Команды с `resume_supported = true` могут работать без активной сессии и используются для загрузки/управления сессиями. Примеры: `/help`, `/status`, `/model`.

## Лучшие практики

1. **Всегда проверяйте ошибки парсинга** перед использованием результата
2. **Используйте `render_slash_command_help()`** для предоставления справки при ошибках
3. **Следуйте существующим паттернам** при добавлении новых команд
4. **Добавляйте тесты** для каждой новой команды
5. **Обновляйте категорию** в `slash_command_category` для правильной группировки

## Лицензия

См. корень репозитория.
