
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <fstream>
#include <sstream>
#include <algorithm>
#include <ctime>
#include <cstdlib>
#include <filesystem>

namespace fs = std::filesystem;

// --- Utility: escape / unescape for '|' delimiter ---
std::string escapeField(const std::string& s) {
    std::string out;
    for (char c : s) {
        if (c == '|' || c == '\\') {
            out.push_back('\\');
        }
        out.push_back(c);
    }
    return out;
}

std::string unescapeField(const std::string& s) {
    std::string out;
    bool esc = false;
    for (size_t i = 0; i < s.size(); ++i) {
        char c = s[i];
        if (esc) {
            out.push_back(c);
            esc = false;
        } else {
            if (c == '\\') esc = true;
            else out.push_back(c);
        }
    }
    return out;
}

std::vector<std::string> splitEscaped(const std::string& line, char delim='|') {
    std::vector<std::string> parts;
    std::string cur;
    bool esc = false;
    for (size_t i = 0; i < line.size(); ++i) {
        char c = line[i];
        if (esc) {
            cur.push_back(c);
            esc = false;
        } else {
            if (c == '\\') {
                esc = true;
            } else if (c == delim) {
                parts.push_back(cur);
                cur.clear();
            } else {
                cur.push_back(c);
            }
        }
    }
    parts.push_back(cur);
    return parts;
}

// --- Character ---
class Character {
    std::string name;
    std::string role;
    int power;
    std::string graphics_url;

public:
    Character() : name(""), role(""), power(1), graphics_url("") {}
    Character(const std::string& n, const std::string& r, int p, const std::string& g)
        : name(n), role(r), power(p), graphics_url(g) {}

    void display() const {
        std::cout << "Ім’я: " << name << "\nРоль: " << role << "\nСила: " << power
                  << "\nГрафіка: " << (graphics_url.empty() ? "(не вказано)" : graphics_url) << "\n";
    }

    std::string getName() const { return name; }
    std::string getRole() const { return role; }
    int getPower() const { return power; }
    std::string getGraphics() const { return graphics_url; }

    void setName(const std::string& n) { name = n; }
    void setRole(const std::string& r) { role = r; }
    void setPower(int p) { power = p; }
    void setGraphics(const std::string& g) { graphics_url = g; }

    std::string serialize() const {
        // name|role|power|graphics_url
        std::ostringstream ss;
        ss << escapeField(name) << "|" << escapeField(role) << "|" << power << "|" << escapeField(graphics_url);
        return ss.str();
    }

    static Character deserialize(const std::string& line) {
        auto parts = splitEscaped(line, '|');
        Character c;
        if (parts.size() >= 4) {
            c.name = unescapeField(parts[0]);
            c.role = unescapeField(parts[1]);
            c.power = std::stoi(parts[2]);
            c.graphics_url = unescapeField(parts[3]);
        }
        return c;
    }
};

// --- CharacterCatalog ---
class CharacterCatalog {
    std::vector<std::shared_ptr<Character>> characters;
    fs::path dataPath;

public:
    CharacterCatalog(const fs::path& p = "data/characters.txt") : dataPath(p) {
        // Ensure data folder exists
        if (dataPath.has_parent_path()) fs::create_directories(dataPath.parent_path());
        loadFromFile();
    }

    void addCharacter(const std::string& name, const std::string& role, int power, const std::string& graphics) {
        characters.push_back(std::make_shared<Character>(name, role, power, graphics));
        saveToFile();
    }

    void listCharacters() const {
        std::cout << "\n--- Список персонажів ---\n";
        if (characters.empty()) {
            std::cout << "(порожньо)\n";
            return;
        }
        for (const auto& c : characters) {
            c->display();
            std::cout << "-------------------------\n";
        }
    }

    std::shared_ptr<Character> selectCharacter(const std::string& name) const {
        for (const auto& c : characters) {
            if (c->getName() == name) return c;
        }
        return nullptr;
    }

    std::shared_ptr<Character> selectByIndex(int idx) const {
        if (idx < 0 || idx >= (int)characters.size()) return nullptr;
        return characters[idx];
    }

    void saveToFile() const {
        std::ofstream ofs(dataPath, std::ios::trunc);
        if (!ofs) {
            std::cerr << "Не вдалося відкрити файл для збереження персонажів: " << dataPath << "\n";
            return;
        }
        for (const auto& c : characters) {
            ofs << c->serialize() << "\n";
        }
    }

    void loadFromFile() {
        characters.clear();
        std::ifstream ifs(dataPath);
        if (!ifs) return;
        std::string line;
        while (std::getline(ifs, line)) {
            if (line.empty()) continue;
            Character c = Character::deserialize(line);
            characters.push_back(std::make_shared<Character>(c));
        }
    }

    void editCharacter(const std::string& name) {
        auto c = selectCharacter(name);
        if (!c) {
            std::cout << "Персонаж не знайдений.\n";
            return;
        }
        std::cout << "Редагування персонажа (" << name << ")\n";
        std::string input;
        std::cout << "Нова роль (залишити порожнім щоб не змінювати, поточна: " << c->getRole() << "): ";
        std::getline(std::cin, input);
        if (!input.empty()) c->setRole(input);
        std::cout << "Нова сила (1-100, поточна: " << c->getPower() << "): ";
        std::getline(std::cin, input);
        if (!input.empty()) {
            try {
                int p = std::stoi(input);
                c->setPower(std::clamp(p, 1, 100));
            } catch (...) {}
        }
        std::cout << "Новий URL графіки (залишити порожнім щоб не змінювати, поточний: " << c->getGraphics() << "): ";
        std::getline(std::cin, input);
        if (!input.empty()) c->setGraphics(input);

        saveToFile();
        std::cout << "Збережено.\n";
    }

    int count() const { return (int)characters.size(); }
    void ensureSample() {
        if (!characters.empty()) return;
        addCharacter("Олександр", "Воїн", 75, "http://example.com/oleksandr.png");
        addCharacter("Марія", "Маг", 60, "http://example.com/maria.png");
        addCharacter("Тарас", "Розвідник", 50, "");
    }
};

// --- Quest & QuestLog (Тема 3) ---
enum class QuestState { Open, Completed };

struct Quest {
    int id;
    std::string title;
    std::string description;
    QuestState state;

    std::string serialize() const {
        // id|title|description|state
        std::ostringstream ss;
        ss << id << "|" << escapeField(title) << "|" << escapeField(description) << "|" << (state == QuestState::Open ? "0" : "1");
        return ss.str();
    }

    static Quest deserialize(const std::string& line) {
        auto parts = splitEscaped(line, '|');
        Quest q;
        if (parts.size() >= 4) {
            q.id = std::stoi(parts[0]);
            q.title = unescapeField(parts[1]);
            q.description = unescapeField(parts[2]);
            q.state = (parts[3] == "0") ? QuestState::Open : QuestState::Completed;
        }
        return q;
    }
};

class QuestLog {
    std::vector<Quest> quests;
    fs::path dataPath;

public:
    QuestLog(const fs::path& p = "data/quests.txt") : dataPath(p) {
        if (dataPath.has_parent_path()) fs::create_directories(dataPath.parent_path());
        load();
    }

    void addQuest(const std::string& title, const std::string& desc) {
        int nextId = 1;
        if (!quests.empty()) nextId = quests.back().id + 1;
        quests.push_back({nextId, title, desc, QuestState::Open});
        save();
    }

    void listQuests() const {
        std::cout << "\n--- Журнал квестів ---\n";
        if (quests.empty()) {
            std::cout << "(порожньо)\n";
            return;
        }
        for (const auto& q : quests) {
            std::cout << "ID: " << q.id << " | " << q.title << " | " << (q.state == QuestState::Open ? "Відкритий" : "Виконаний") << "\n";
            std::cout << "Опис: " << q.description << "\n";
            std::cout << "-----------------\n";
        }
    }

    void completeQuest(int id) {
        for (auto& q : quests) {
            if (q.id == id) {
                q.state = QuestState::Completed;
                save();
                std::cout << "Квест #" << id << " позначено як виконаний.\n";
                return;
            }
        }
        std::cout << "Квест з таким ID не знайдено.\n";
    }

    void save() const {
        std::ofstream ofs(dataPath, std::ios::trunc);
        if (!ofs) return;
        for (const auto& q : quests) ofs << q.serialize() << "\n";
    }

    void load() {
        quests.clear();
        std::ifstream ifs(dataPath);
        if (!ifs) return;
        std::string line;
        while (std::getline(ifs, line)) {
            if (line.empty()) continue;
            quests.push_back(Quest::deserialize(line));
        }
    }
};

// --- GameSession ---
class GameSession {
    std::shared_ptr<Character> character;
    int turn;
    int score;
    fs::path savePath;

public:
    GameSession(std::shared_ptr<Character> ch) : character(ch), turn(0), score(0) {
        // build save path
        std::string safeName = character->getName();
        // replace spaces with _
        for (auto& c : safeName) if (c == ' ') c = '_';
        savePath = fs::path("data") / ("game_state_" + safeName + ".sav");
        load();
    }

    void start() {
        std::srand((unsigned)std::time(nullptr));
        std::cout << "\n---- Початок гри для " << character->getName() << " ----\n";
        bool running = true;
        while (running) {
            std::cout << "\nХід: " << turn << " | Очки: " << score << " | Сила: " << character->getPower() << "\n";
            std::cout << "1) Пригоди (випадкова подія)\n";
            std::cout << "2) Відпочити (+силу)\n";
            std::cout << "3) Статус персонажа\n";
            std::cout << "4) Зберегти гру\n";
            std::cout << "0) Вийти в меню (автозбереження)\n";
            std::cout << "Ваш вибір: ";
            int ch;
            if (!(std::cin >> ch)) {
                std::cin.clear();
                std::cin.ignore(10000, '\n');
                continue;
            }
            std::cin.ignore();
            switch (ch) {
            case 1: adventureEvent(); break;
            case 2: rest(); break;
            case 3: character->display(); break;
            case 4: save(); break;
            case 0: save(); running = false; break;
            default: std::cout << "Невірний варіант\n"; break;
            }
        }
        std::cout << "Вихід з гри...\n";
    }

    void adventureEvent() {
        ++turn;
        int r = std::rand() % 100;
        if (r < 50) {
            int gained = (std::rand() % 10) + 1;
            score += gained;
            std::cout << "Ти знайшов скарб! + " << gained << " очок.\n";
        } else if (r < 80) {
            int lost = (std::rand() % 5) + 1;
            score = std::max(0, score - lost);
            std::cout << "Під час пригоди втратив трохи очок: - " << lost << ".\n";
        } else {
            int powChange = (std::rand() % 7) - 3; // -3..+3
            int newPow = std::clamp(character->getPower() + powChange, 1, 100);
            character->setPower(newPow);
            std::cout << "Сильна подія вплинула на твою силу: зараз " << newPow << ".\n";
        }
    }

    void rest() {
        ++turn;
        int inc = (std::rand() % 5) + 1;
        character->setPower(std::clamp(character->getPower() + inc, 1, 100));
        std::cout << "Відпочинок дали + " << inc << " сили.\n";
    }

    void save() {
        std::ofstream ofs(savePath, std::ios::trunc);
        if (!ofs) {
            std::cout << "Не вдалося зберегти стан гри.\n";
            return;
        }
        // формат: name|turn|score|power
        ofs << escapeField(character->getName()) << "|" << turn << "|" << score << "|" << character->getPower() << "\n";
        std::cout << "Гру збережено у " << savePath << "\n";
    }

    void load() {
        std::ifstream ifs(savePath);
        if (!ifs) return;
        std::string line;
        if (!std::getline(ifs, line)) return;
        auto parts = splitEscaped(line, '|');
        if (parts.size() >= 4) {
            // name check omitted
            try {
                turn = std::stoi(parts[1]);
                score = std::stoi(parts[2]);
                int p = std::stoi(parts[3]);
                character->setPower(p);
            } catch (...) {}
        }
    }
};

// --- Menus ---
void showMainMenu() {
    std::cout << "\n=== Головне меню ===\n";
    std::cout << "1. Каталог персонажів\n";
    std::cout << "2. Грати\n";
    std::cout << "3. Журнал квестів (Тема 3)\n";
    std::cout << "0. Вихід\n";
}

void characterMenu(CharacterCatalog& catalog) {
    while (true) {
        std::cout << "\n--- Меню каталогу ---\n";
        std::cout << "1) Додати персонажа\n";
        std::cout << "2) Показати всіх персонажів\n";
        std::cout << "3) Вибрати персонажа для перегляду\n";
        std::cout << "4) Редагувати персонажа\n";
        std::cout << "0) Повернутись\n";
        std::cout << "Ваш вибір: ";
        int choice;
        if (!(std::cin >> choice)) {
            std::cin.clear();
            std::cin.ignore(10000, '\n');
            continue;
        }
        std::cin.ignore();
        if (choice == 1) {
            std::string name, role, graphics;
            int power;
            std::cout << "Ім’я персонажа: ";
            std::getline(std::cin, name);
            std::cout << "Роль персонажа: ";
            std::getline(std::cin, role);
            std::cout << "Сила персонажа (1–100): ";
            std::cin >> power;
            std::cin.ignore();
            std::cout << "Посилання на графіку (URL) (можна лишити порожнім): ";
            std::getline(std::cin, graphics);
            catalog.addCharacter(name, role, std::clamp(power, 1, 100), graphics);
            std::cout << "Додано.\n";
        } else if (choice == 2) {
            catalog.listCharacters();
        } else if (choice == 3) {
            if (catalog.count() == 0) {
                std::cout << "Каталог порожній.\n";
                continue;
            }
            std::cout << "Виберіть персонажа за індексом:\n";
            for (int i = 0; i < catalog.count(); ++i) {
                auto c = catalog.selectByIndex(i);
                std::cout << i << ") " << c->getName() << " (" << c->getRole() << ")\n";
            }
            std::cout << "Індекс: ";
            int idx;
            if (!(std::cin >> idx)) {
                std::cin.clear();
                std::cin.ignore(10000, '\n');
                continue;
            }
            std::cin.ignore();
            auto sel = catalog.selectByIndex(idx);
            if (sel) {
                std::cout << "\nОбраний персонаж:\n";
                sel->display();
            } else {
                std::cout << "Неправильний індекс.\n";
            }
        } else if (choice == 4) {
            std::cout << "Введіть ім'я персонажа для редагування: ";
            std::string name;
            std::getline(std::cin, name);
            catalog.editCharacter(name);
        } else if (choice == 0) {
            break;
        } else {
            std::cout << "Невірний вибір.\n";
        }
    }
}

void playMenu(CharacterCatalog& catalog) {
    if (catalog.count() == 0) {
        std::cout << "Каталог порожній. Додайте персонажів перед початком гри.\n";
        return;
    }
    std::cout << "\n--- Вибір персонажа для гри ---\n";
    for (int i = 0; i < catalog.count(); ++i) {
        auto c = catalog.selectByIndex(i);
        std::cout << i << ") " << c->getName() << " (" << c->getRole() << ")\n";
    }
    std::cout << "Індекс персонажа (або -1 щоб повернутися): ";
    int idx;
    if (!(std::cin >> idx)) {
        std::cin.clear();
        std::cin.ignore(10000, '\n');
        return;
    }
    std::cin.ignore();
    if (idx < 0) return;
    auto sel = catalog.selectByIndex(idx);
    if (!sel) {
        std::cout << "Невірний індекс.\n";
        return;
    }
    GameSession session(sel);
    session.start();
}

void questMenu(QuestLog& qlog) {
    while (true) {
        std::cout << "\n--- Журнал квестів ---\n";
        std::cout << "1) Додати квест\n";
        std::cout << "2) Показати квести\n";
        std::cout << "3) Позначити квест як виконаний\n";
        std::cout << "0) Повернутись\n";
        std::cout << "Ваш вибір: ";
        int ch;
        if (!(std::cin >> ch)) {
            std::cin.clear();
            std::cin.ignore(10000, '\n');
            continue;
        }
        std::cin.ignore();
        if (ch == 1) {
            std::string title, desc;
            std::cout << "Назва квесту: ";
            std::getline(std::cin, title);
            std::cout << "Опис: ";
            std::getline(std::cin, desc);
            qlog.addQuest(title, desc);
            std::cout << "Додано.\n";
        } else if (ch == 2) {
            qlog.listQuests();
        } else if (ch == 3) {
            std::cout << "Введіть ID квесту для завершення: ";
            int id;
            if (!(std::cin >> id)) {
                std::cin.clear();
                std::cin.ignore(10000, '\n');
                continue;
            }
            std::cin.ignore();
            qlog.completeQuest(id);
        } else if (ch == 0) {
            break;
        } else {
            std::cout << "Невірний вибір.\n";
        }
    }
}

int main() {
    CharacterCatalog catalog;
    catalog.ensureSample();
    QuestLog qlog;
    bool running = true;
    while (running) {
        showMainMenu();
        std::cout << "Ваш вибір: ";
        int choice;
        if (!(std::cin >> choice)) {
            std::cin.clear();
            std::cin.ignore(10000, '\n');
            continue;
        }
        std::cin.ignore();
        if (choice == 1) {
            characterMenu(catalog);
        } else if (choice == 2) {
            playMenu(catalog);
        } else if (choice == 3) {
            questMenu(qlog);
        } else if (choice == 0) {
            std::cout << "Завершення програми...\n";
            running = false;
        } else {
            std::cout << "Невірний вибір.\n";
        }
    }
    return 0;
}