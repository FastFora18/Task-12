## Практическая работа № 12
```Csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
namespace NotificationGateway
    // part 1
{
    public class EmailChannel : NotificationChannel
    {
        private readonly bool _smtpAvailable;

        public override bool IsOnline => _smtpAvailable;

        public EmailChannel(string name, bool smtpAvailable = true) : base(name)
        {
            _smtpAvailable = smtpAvailable;
        }

        public override string FormatPayload(string rawMessage)
        {
            // Вызов базового форматирования + почтовый колонтитул
            return base.FormatPayload(rawMessage) + "\n---\nС уважением, Почтовый Сервис";
        }

        public override bool Deliver(string recipient, string message)
        {
            if (!IsOnline)
            {
                Console.WriteLine($"[{ChannelName}] SMTP-сервер недоступен.");
                return false;
            }

            if (string.IsNullOrEmpty(recipient) || !recipient.Contains('@'))
            {
                Console.WriteLine($"[{ChannelName}] Некорректный адрес: '{recipient}'. Отправка отменена.");
                return false;
            }

            DeliveredCount++;
            Console.WriteLine($"[{ChannelName}] -> {recipient}");
            Console.WriteLine(FormatPayload(message));
            Console.WriteLine();
            return true;
        }
    }
}
// part 2
namespace NotificationGateway
{
    // Абстрактный базовый класс — контракт для всех каналов уведомлений.
    public abstract class NotificationChannel
    {
        protected string ChannelName;
        public int DeliveredCount { get; protected set; }

        // Абстрактное свойство — обязательная реализация у потомков
        public abstract bool IsOnline { get; }

        protected NotificationChannel(string name)
        {
            ChannelName = name;
        }

        // Абстрактный метод отправки
        public abstract bool Deliver(string recipient, string message);

        // Виртуальный метод форматирования — может быть переопределён
        public virtual string FormatPayload(string rawMessage)
        {
            return $"[{ChannelName}] {rawMessage}";
        }

        // Обычный метод отчёта (общий для всех)
        public void PrintAudit()
        {
            Console.WriteLine($"[AUDIT] Канал '{ChannelName}': доставлено {DeliveredCount} оповещ.");
        }
    }
}
// part 3
namespace NotificationGateway
{
    internal class Program
    {
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.WriteLine("=== Мультиканальный шлюз уведомлений и тревог ===\n");

            // === Шаг 5.1. Полиморфный массив ===
            NotificationChannel[] channels = new NotificationChannel[3];
            channels[0] = new EmailChannel("Email SMTP", smtpAvailable: true);
            channels[1] = new TelegramChannel("Telegram Bot", botToken: "12345:ABC");
            channels[2] = new SirenAlarmChannel(soundRelayLevel: 90);

            // === Шаг 5.2. Демонстрация позднего связывания (override) ===
            Console.WriteLine("--- Демонстрация полиморфного вызова Deliver (override) ---\n");

            string testMsg = "Внимание: регламентные технические работы в серверной!";

            channels[0].Deliver("admin@company.ru", testMsg);      // Email: валидный адрес
            channels[1].Deliver("@sys_admin", testMsg);            // Telegram: валидный юзернейм
            channels[2].Deliver("ANY", testMsg);                   // Сирена: получатель игнорируется

            // Демонстрация неудачных отправок (валидация)
            Console.WriteLine("--- Проверка валидации ---\n");
            channels[0].Deliver("invalid-address", "тест");        // нет @
            channels[1].Deliver("sys_admin", "тест");              // нет @
            channels[1].Deliver("@user", new string('X', 150));    // > 100 символов

            // === Шаг 5.3. Разница между override и new ===
            Console.WriteLine("\n--- Демонстрация override vs new ---\n");

            SirenAlarmChannel sirenDirect = (SirenAlarmChannel)channels[2];
            NotificationChannel sirenAsBase = channels[2];

            // Через ссылку базового типа — сработает ВИРТУАЛЬНЫЙ метод базового класса
            Console.WriteLine("Через ссылку базового типа NotificationChannel:");
            Console.WriteLine("  " + sirenAsBase.FormatPayload("тест сокрытия"));

            // Через прямую ссылку SirenAlarmChannel — сработает метод, скрытый через new
            Console.WriteLine("\nЧерез прямую ссылку SirenAlarmChannel:");
            Console.WriteLine("  " + sirenDirect.FormatPayload("тест сокрытия"));

            // === Шаг 5.4. Полиморфный аудит ===
            Console.WriteLine("\n--- Итоговый аудит по каналам ---\n");
            foreach (var channel in channels)
            {
                channel.PrintAudit();
            }

            Console.WriteLine("\nНажмите любую клавишу для выхода...");
            Console.ReadKey();
        }
    }
}
// part 4
namespace NotificationGateway
{
    public class SirenAlarmChannel : NotificationChannel
    {
        private readonly int _soundRelayLevel;

        public override bool IsOnline => _soundRelayLevel > 50;

        public SirenAlarmChannel(int soundRelayLevel = 80) : base("Звуковая Сирена")
        {
            _soundRelayLevel = soundRelayLevel;
        }

        // СОКРЫТИЕ (new) базового виртуального метода
        public new string FormatPayload(string rawMessage)
        {
            return "[ТРЕВОГА МЧС]: " + rawMessage.ToUpper();
        }

        public override bool Deliver(string recipient, string message)
        {
            if (!IsOnline)
            {
                Console.WriteLine($"[{ChannelName}] Уровень реле ниже порога ({_soundRelayLevel}). Сирена не сработала.");
                return false;
            }

            DeliveredCount++;
            Console.WriteLine($"[{ChannelName}] !!! ТРЕВОГА !!!");
            Console.WriteLine(">>> \a ВНИМАНИЕ! СИРЕНА АКТИВИРОВАНА \a <<<");
            Console.WriteLine(message.ToUpper());
            Console.WriteLine();
            return true;
        }
    }
}
// part 5
namespace NotificationGateway
{
    public class TelegramChannel : NotificationChannel
    {
        private readonly string _botToken;

        public override bool IsOnline => true;

        public TelegramChannel(string name, string botToken) : base(name)
        {
            _botToken = botToken;
        }

        public override bool Deliver(string recipient, string message)
        {
            if (!IsOnline)
            {
                Console.WriteLine($"[{ChannelName}] Бот оффлайн.");
                return false;
            }

            if (string.IsNullOrEmpty(recipient) || !recipient.StartsWith("@"))
            {
                Console.WriteLine($"[{ChannelName}] Получатель должен начинаться с '@': '{recipient}'");
                return false;
            }

            if (message.Length > 100)
            {
                Console.WriteLine($"[{ChannelName}] Сообщение длиннее 100 символов ({message.Length}). Отправка отменена.");
                return false;
            }

            DeliveredCount++;
            Console.WriteLine($"[{ChannelName}] *{recipient}* (символов: {message.Length})");
            Console.WriteLine(FormatPayload($"*{message}*"));
            Console.WriteLine();
            return true;
        }
    }
}
```
## Результат выполнения:
<img width="1458" height="852" alt="1" src="https://github.com/user-attachments/assets/115c0bc6-70b8-49fd-b6fe-5fb4fa98dde3" />
<img width="1462" height="857" alt="2" src="https://github.com/user-attachments/assets/e9923ffc-f1a2-49f8-85c2-8170e2d38137" />
<img width="1458" height="806" alt="3" src="https://github.com/user-attachments/assets/c85a872e-3b73-4749-a328-2c4c38f06257" />
<img width="1460" height="849" alt="4" src="https://github.com/user-attachments/assets/8cbf2966-f973-4266-ab06-530013f9ee98" />
<img width="1459" height="853" alt="5" src="https://github.com/user-attachments/assets/aedc44c3-8c9a-429f-9ffe-8ecd0a622d28" />

// Делал Аристов Максим
