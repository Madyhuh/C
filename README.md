#include <QApplication>
#include <QWidget>
#include <QLabel>
#include <QRadioButton>
#include <QPushButton>
#include <QButtonGroup>
#include <QVBoxLayout>
#include <QMessageBox>
#include <QFile>
#include <QTextStream>
#include <QList>
#include <QMap>
#include <QDebug>
#include <QPair>
#include <algorithm>
#include <QInputDialog>

struct Question {
    QString text;
    QStringList options;
    int correctAnswer;
};

class QuizWidget : public QWidget {
    Q_OBJECT

public:
    QuizWidget(QWidget *parent = nullptr) : QWidget(parent), currentQuestion(0), score(0) {
        setWindowTitle("India States Quiz");

        bool ok;
        playerName = QInputDialog::getText(this, "Welcome to the Quiz", "Enter your name:", QLineEdit::Normal, "", &ok);
        if (!ok || playerName.isEmpty()) {
            playerName = "Player";
        }

        questionLabel = new QLabel();
        submitButton = new QPushButton("Submit");
        buttonGroup = new QButtonGroup();

        QVBoxLayout *mainLayout = new QVBoxLayout(this);
        mainLayout->addWidget(questionLabel);

        for (int i = 0; i < 4; ++i) {
            optionButtons.append(new QRadioButton());
            buttonGroup->addButton(optionButtons[i]);
            mainLayout->addWidget(optionButtons[i]);
        }

        mainLayout->addWidget(submitButton);

        connect(submitButton, &QPushButton::clicked, this, &QuizWidget::checkAnswer);

        if (!loadQuestions("questions.txt")) {
            QMessageBox::critical(this, "Error", "Failed to load questions!");
            exit(1);
        }

        loadLeaderboard("leaderboard.txt");

        showQuestion();
    }

private slots:
    void checkAnswer() {
        int selectedOption = -1;
        for (int i = 0; i < optionButtons.size(); ++i) {
            if (optionButtons[i]->isChecked()) {
                selectedOption = i;
                break;
            }
        }

        if (selectedOption == -1) {
            QMessageBox::warning(this, "Warning", "Please select an answer.");
            return;
        }

        if (selectedOption == questions[currentQuestion].correctAnswer) {
            score++;
            QMessageBox::information(this, "Correct", "Correct!");
        } else {
            QMessageBox::information(this, "Incorrect", "Incorrect!");
        }

        currentQuestion++;
        if (currentQuestion < questions.size()) {
            showQuestion();
        } else {
            endQuiz();
        }
    }

private:
    bool loadQuestions(const QString &filename) {
        QFile file(filename);
        if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
            qDebug() << "Error opening questions file:" << file.errorString();
            return false;
        }

        QTextStream in(&file);
        while (!in.atEnd()) {
            QString line = in.readLine();
            QStringList parts = line.split("|");
            if (parts.size() >= 6) {
                Question q;
                q.text = parts[0];
                q.options = parts.mid(1, 4);
                q.correctAnswer = parts[5].toInt();
                questions.append(q);
            } else {
                qDebug() << "Invalid question format:" << line;
            }
        }
        file.close();
        return true;
    }

    void showQuestion() {
        if (currentQuestion < questions.size()) {
            questionLabel->setText(questions[currentQuestion].text);
            for (int i = 0; i < optionButtons.size(); ++i) {
                if (i < questions[currentQuestion].options.size()) {
                    optionButtons[i]->setText(questions[currentQuestion].options[i]);
                    optionButtons[i]->setVisible(true);
                } else {
                    optionButtons[i]->setVisible(false);
                }
                optionButtons[i]->setChecked(false);
            }
        }
    }

    void endQuiz() {
        if (leaderboard.contains(playerName)) {
            if (score > leaderboard[playerName]) {
                leaderboard[playerName] = score;
            }
        } else {
            leaderboard[playerName] = score;
        }
        saveLeaderboard("leaderboard.txt");

        QString message = QString("Quiz finished!\nYour score: %1\nLeaderboard:\n").arg(score);

        QList<QPair<QString, int>> sortedLeaderboard;
        for (auto it = leaderboard.begin(); it != leaderboard.end(); ++it) {
            sortedLeaderboard.append(qMakePair(it.key(), it.value()));
        }
        std::sort(sortedLeaderboard.begin(), sortedLeaderboard.end(), [](const QPair<QString, int>& a, const QPair<QString, int>& b) {
            return a.second > b.second;
        });

        for (const auto& pair : sortedLeaderboard) {
            message += QString("%1: %2\n").arg(pair.first).arg(pair.second);
        }

        QMessageBox::information(this, "Quiz Finished", message);

        currentQuestion = 0;
        score = 0;
        showQuestion();
    }

    void loadLeaderboard(const QString &filename) {
        QFile file(filename);
        if (file.open(QIODevice::ReadOnly | QIODevice::Text)) {
            QTextStream in(&file);
            while (!in.atEnd()) {
                QString line = in.readLine();
                QStringList parts = line.split(":");
                if (parts.size() == 2) {
                    leaderboard[parts[0]] = parts[1].toInt();
                }
            }
            file.close();
        }
    }

    void saveLeaderboard(const QString &filename) {
        QFile file(filename);
        if (file.open(QIODevice::WriteOnly | QIODevice::Text)) {
            QTextStream out(&file);
            for (auto it = leaderboard.begin(); it != leaderboard.end(); ++it) {
                out << it.key() << ":" << it.value() << "\n";
            }
            file.close();
        }
    }

    QLabel *questionLabel;
    QList<QRadioButton*> optionButtons;
    QPushButton *submitButton;
    QButtonGroup *buttonGroup;
    QList<Question> questions;
    int currentQuestion;
    int score;
    QMap<QString, int> leaderboard;
    QString playerName;
};

int main(int argc, char *argv[]) {
    QApplication a(argc, argv);
    QuizWidget w;
    w.show();
    return a.exec();
}

#include "main.moc"
