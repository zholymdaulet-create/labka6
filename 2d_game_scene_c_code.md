#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>

#include <iostream>
#include <vector>
#include <cmath>

// Терезе өлшемдері
const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600;

// Ойын күйі (Пауза үшін Space пернесі)
bool isPaused = false;
bool spacePressedLastFrame = false;

// Шейдерлер
const char* vertexShaderSource = "#version 330 core\n"
"layout (location = 0) in vec3 aPos;\n"
"uniform mat4 transform;\n"
"void main()\n"
"{\n"
"   gl_Position = transform * vec4(aPos, 1.0);\n"
"}\0";

const char* fragmentShaderSource = "#version 330 core\n"
"out vec4 FragColor;\n"
"uniform vec4 ourColor;\n"
"void main()\n"
"{\n"
"   FragColor = ourColor;\n"
"}\0";

// Пернетақта енгізуін өңдеу
void processInput(GLFWwindow *window) {
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);

    bool spaceCurrent = glfwGetKey(window, GLFW_KEY_SPACE) == GLFW_PRESS;
    if (spaceCurrent && !spacePressedLastFrame) {
        isPaused = !isPaused;
    }
    spacePressedLastFrame = spaceCurrent;
}

void framebuffer_size_callback(GLFWwindow* window, int width, int height) {
    glViewport(0, 0, width, height);
}

int main() {
    if (!glfwInit()) return -1;
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

    GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "2D Game Scene - OpenGL", NULL, NULL);
    if (window == NULL) {
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);
    glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

    glewExperimental = GL_TRUE;
    if (glewInit() != GLEW_OK) return -1;

    // Шейдерлерді компиляциялау
    unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
    glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
    glCompileShader(vertexShader);

    unsigned int fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
    glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL);
    glCompileShader(fragmentShader);

    unsigned int shaderProgram = glCreateProgram();
    glAttachShader(shaderProgram, vertexShader);
    glAttachShader(shaderProgram, fragmentShader);
    glLinkProgram(shaderProgram);

    glDeleteShader(vertexShader);
    glDeleteShader(fragmentShader);

    // ==========================================
    // 1. ШАРДЫ ЖАСАУ (EBO арқылы: Центр + 48 нүкте)
    // ==========================================
    float radius = 0.1f; // Шар радиусы
    int segments = 48;    // Нүктелер саны
    
    std::vector<float> ballVertices;
    ballVertices.push_back(0.0f); // Центр (0 индекс)
    ballVertices.push_back(0.0f);
    ballVertices.push_back(0.0f);

    for (int i = 0; i < segments; i++) {
        float theta = 2.0f * 3.14159265359f * float(i) / float(segments);
        ballVertices.push_back(radius * cosf(theta));
        ballVertices.push_back(radius * sinf(theta));
        ballVertices.push_back(0.0f);
    }

    std::vector<unsigned int> ballIndices;
    for (int i = 1; i <= segments; i++) {
        ballIndices.push_back(0); 
        ballIndices.push_back(i);
        ballIndices.push_back((i % segments) + 1);
    }

    unsigned int ballVAO, ballVBO, ballEBO;
    glGenVertexArrays(1, &ballVAO);
    glGenBuffers(1, &ballVBO);
    glGenBuffers(1, &ballEBO);

    glBindVertexArray(ballVAO);
    glBindBuffer(GL_ARRAY_BUFFER, ballVBO);
    glBufferData(GL_ARRAY_BUFFER, ballVertices.size() * sizeof(float), ballVertices.data(), GL_STATIC_DRAW);

    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ballEBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, ballIndices.size() * sizeof(unsigned int), ballIndices.data(), GL_STATIC_DRAW);

    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);

    // ==========================================
    // 2. ШАРШЫНЫ ЖАСАУ (Square)
    // ==========================================
    float sqSize = 0.08f; // Шаршының өлшемі шардан кішірек етіліп, соқтығысу болмас үшін алынды
    float squareVertices[] = {
        -sqSize, -sqSize, 0.0f,
         sqSize, -sqSize, 0.0f,
         sqSize,  sqSize, 0.0f,

         sqSize,  sqSize, 0.0f,
        -sqSize,  sqSize, 0.0f,
        -sqSize, -sqSize, 0.0f
    };

    unsigned int squareVAO, squareVBO;
    glGenVertexArrays(1, &squareVAO);
    glGenBuffers(1, &squareVBO);

    glBindVertexArray(squareVAO);
    glBindBuffer(GL_ARRAY_BUFFER, squareVBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(squareVertices), squareVertices, GL_STATIC_DRAW);

    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);

    // Уақыт айнымалылары
    float deltaTime = 0.0f;
    float lastFrame = 0.0f;
    float gameTime = 0.0f;

    float ballPosX = 0.0f; // Шар қатаң түрде ортада тұрады

    // Негізгі ойын циклі
    while (!glfwWindowShouldClose(window)) {
        float currentFrame = (float)glfwGetTime();
        deltaTime = currentFrame - lastFrame;
        lastFrame = currentFrame;

        if (!isPaused) {
            gameTime += deltaTime;
        }

        processInput(window);

        // Экранды тазарту
        glClearColor(0.1f, 0.1f, 0.15f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        glUseProgram(shaderProgram);

        // ==========================================
        // 3. ШАРДЫҢ СЕКІРУІ (Тік бағытта, бір сызық бойымен)
        // ==========================================
        float bouncePeriod = 1.8f; // Секіру уақыты (секунд)
        float ballPhase = fmod(gameTime, bouncePeriod) / bouncePeriod;
        
        // Шардың биіктігі (жер деңгейі -0.4f, жоғары көтерілуі 0.6f)
        float ballPosY = -0.4f + fabsf(sinf(ballPhase * 3.14159265359f)) * 0.6f; 

        glm::mat4 ballModel = glm::mat4(1.0f);
        ballModel = glm::translate(ballModel, glm::vec3(ballPosX, ballPosY, 0.0f));

        int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
        glUniform4f(vertexColorLocation, 1.0f, 0.5f, 0.2f, 1.0f); // Қызғылт сары шар

        int transformLoc = glGetUniformLocation(shaderProgram, "transform");
        glUniformMatrix4fv(transformLoc, 1, GL_FALSE, glm::value_ptr(ballModel));

        glBindVertexArray(ballVAO);
        glDrawElements(GL_TRIANGLES, ballIndices.size(), GL_UNSIGNED_INT, 0);

        // ==========================================
        // 4. ШАРШЫНЫҢ СЫРҒУЫ (Солдан оңға және оңнан солға, қауіпсіз тайминг)
        // ==========================================
        float sqPeriod = 3.6f; // Шаршының бір барып қайту уақыты
        float sqPhase = fmod(gameTime, sqPeriod) / sqPeriod;
        
        // Шаршының X координатасы (-1.3 пен 1.3 аралығында сырғиды)
        // Cosine қолданылды, өйткені шаршы ортадан (X=0) өткенде шар ең биік нүктесінде болады
        float sqPosX = cosf(sqPhase * 2.0f * 3.14159265359f) * 1.3f;

        glm::mat4 sqModel = glm::mat4(1.0f);
        // Шаршы шардың астынан өтетіндей төменгі сызық бойымен сырғиды (Y = -0.4f)
        // Ескерту: glm::rotate мүлде жоқ, сондықтан шаршы ешқашан айналмайды, тек бір қалыпта сырғиды!
        sqModel = glm::translate(sqModel, glm::vec3(sqPosX, -0.4f + sqSize, 0.0f)); 

        glUniform4f(vertexColorLocation, 0.2f, 0.8f, 1.0f, 1.0f); // Көгілдір шаршы
        glUniformMatrix4fv(transformLoc, 1, GL_FALSE, glm::value_ptr(sqModel));

        glBindVertexArray(squareVAO);
        glDrawArrays(GL_TRIANGLES, 0, 6);

        glfwSwapBuffers(window);
        glfwPollEvents();
    }

    // Ресурстарды тазарту
    glDeleteVertexArrays(1, &ballVAO);
    glDeleteBuffers(1, &ballVBO);
    glDeleteBuffers(1, &ballEBO);
    glDeleteVertexArrays(1, &squareVAO);
    glDeleteBuffers(1, &squareVBO);
    glDeleteProgram(shaderProgram);

    glfwTerminate();
    return 0;
}