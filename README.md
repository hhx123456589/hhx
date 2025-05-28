# hhx
# 文件结构
java-ansible-dashboard/
├── backend/
│   ├── src/
│   │   └── main/
│   │       ├── java/com/ansibledashboard/
│   │       │   ├── AnsibleDashboardApplication.java
│   │       │   └── AnsibleController.java
│   │       └── resources/
│   │           └── application.properties
│   └── Dockerfile
├── frontend/
│   └── index.html
├── ansible/
│   ├── hosts
│   └── ansible.cfg
└── docker-compose.yml

# Docker配置 (docker-compose.yml)
version: '3.8'

services:
  backend:
    build: ./backend
    container_name: ansible_java_backend
    volumes:
      - ./ansible:/etc/ansible
    ports:
      - "8080:8080"
    privileged: true  # 允许执行ansible命令

  frontend:
    image: nginx:alpine
    container_name: ansible_java_frontend
    volumes:
      - ./frontend:/usr/share/nginx/html
    ports:
      - "80:80"
    depends_on:
      - backend
      
# Java后端实现
# 主应用类 (backend/src/main/java/com/ansibledashboard/AnsibleDashboardApplication.java)     

      package com.ansibledashboard;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;

@SpringBootApplication
public class AnsibleDashboardApplication {

    private static final String ANSIBLE_HOSTS_FILE = "/etc/ansible/hosts";
    private static final String ANSIBLE_CONFIG_FILE = "/etc/ansible/ansible.cfg";

    public static void main(String[] args) {
        // 确保ansible配置存在
        ensureAnsibleConfig();
        SpringApplication.run(AnsibleDashboardApplication.class, args);
    }

    private static void ensureAnsibleConfig() {
        try {
            if (!Files.exists(Paths.get(ANSIBLE_CONFIG_FILE))) {
                Files.write(Paths.get(ANSIBLE_CONFIG_FILE), 
                    "[defaults]\nhost_key_checking = False\n".getBytes());
            }

            if (!Files.exists(Paths.get(ANSIBLE_HOSTS_FILE))) {
                Files.write(Paths.get(ANSIBLE_HOSTS_FILE), 
                    "[all]\nlocalhost ansible_connection=local\n".getBytes());
            }
        } catch (IOException e) {
            System.err.println("Error creating ansible config: " + e.getMessage());
        }
    }
}

# 控制器类 (backend/src/main/java/com/ansibledashboard/AnsibleController.java)
package com.ansibledashboard;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class AnsibleController {

    @GetMapping("/ping")
    public Map<String, Object> ansiblePing() {
        Map<String, Object> response = new HashMap<>();
        try {
            Process process = new ProcessBuilder()
                .command("ansible", "all", "-m", "ping")
                .redirectErrorStream(true)
                .start();

            BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream()));
            StringBuilder output = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                output.append(line).append("\n");
            }

            int exitCode = process.waitFor();
            if (exitCode == 0) {
                response.put("status", "success");
                response.put("output", output.toString());
            } else {
                response.put("status", "error");
                response.put("error", output.toString());
            }
        } catch (IOException | InterruptedException e) {
            response.put("status", "error");
            response.put("error", e.getMessage());
        }
        return response;
    }
}

# 应用配置 (backend/src/main/resources/application.properties)
server.port=8080

# 后端Dockerfile (backend/Dockerfile)
# 构建阶段
FROM maven:3.8.4-openjdk-11 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# 运行阶段
FROM openjdk:11-jre-slim
WORKDIR /app

# 安装ansible和必要工具
RUN apt-get update && apt-get install -y \
    ansible \
    sshpass \
    && rm -rf /var/lib/apt/lists/*

COPY --from=build /app/target/*.jar app.jar
COPY ansible /etc/ansible

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]

# 前端页面 (frontend/index.html)
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Java Ansible Dashboard</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        button {
            padding: 10px 15px;
            background: #4CAF50;
            color: white;
            border: none;
            cursor: pointer;
        }
        pre {
            background: #f4f4f4;
            padding: 10px;
            border-radius: 5px;
            overflow-x: auto;
        }
        .success {
            color: green;
        }
        .error {
            color: red;
        }
    </style>
</head>
<body>
    <h1>Java Ansible Dashboard</h1>
    <button onclick="runPingTest()">Run Ansible Ping</button>
    <div id="result"></div>

    <script>
        async function runPingTest() {
            const resultDiv = document.getElementById('result');
            resultDiv.innerHTML = '<p>Running ansible ping test...</p>';
            
            try {
                const response = await fetch('http://localhost:8080/api/ping');
                const data = await response.json();
                
                let resultHTML = `<h2 class="${data.status}">Result: ${data.status}</h2>`;
                if (data.output) {
                    resultHTML += `<pre>${data.output}</pre>`;
                }
                if (data.error) {
                    resultHTML += `<pre class="error">${data.error}</pre>`;
                }
                
                resultDiv.innerHTML = resultHTML;
            } catch (error) {
                resultDiv.innerHTML = `<p class="error">Failed to connect to backend: ${error.message}</p>`;
            }
        }
    </script>
</body>
</html>

# Ansible基础配置
 # ansible/hosts
 [all]
localhost ansible_connection=local
 # ansible/ansible.cfg
[defaults]
host_key_checking = False

# 构建并运行系统
 # 进入项目目录
 cd java-ansible-dashboard

 # 构建并启动容器
 docker-compose up --build

# 访问系统
 前端访问: http://localhost

 后端API: http://localhost:8080/api/ping
