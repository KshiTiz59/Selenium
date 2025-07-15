# Selenium
  
# index.html
<!DOCTYPE html>
<html>
<head>
    <title>Selenium Test with WebSocket</title>
</head>
<body>
    <h2>Run Selenium Automation</h2>
    <input type="text" id="urlInput" placeholder="Enter URL" />
    <button onclick="startTest()">Run</button>

    <pre id="output"
         style="background:#f4f4f4; padding:10px; height:300px; overflow:auto; display:none;"></pre>

    <script>
        let socketId = "";

        function startTest() {
            const url = document.getElementById('urlInput').value;
            const output = document.getElementById('output');
            output.textContent = "";
            output.style.display = "block";
           
            const clientId = "client-" + Date.now();  
            const socket = new WebSocket("ws://localhost:8080/ws/logs?clientId=" + clientId);
            socket.onopen = () => {
                socketId = socket.url.split("/").pop();  // fallback if needed
                output.textContent += "✅ WebSocket connected.\n";

                // Send POST to backend with URL + socketId
                fetch("http://localhost:8080/run", {
                    method: "POST",
                    headers: { "Content-Type": "application/json" },
                    body: JSON.stringify({ url: url, socketId: clientId })
                }).then(res => res.text()).then(msg => {
                    output.textContent += msg + "\n";
                });
            };

            socket.onmessage = (event) => {
                output.textContent += event.data + "\n";
                output.scrollTop = output.scrollHeight;
            };

            socket.onclose = () => {
                output.textContent += "\n🔌 WebSocket closed.";
            };
        }
    </script>
</body>
</html>

# automationcontroller.java
package com.example.demo.Controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;

import com.example.demo.Hello;
import com.example.demo.WebSocketSessionManager;

import java.io.IOException;
import java.util.Map;

@RestController
@CrossOrigin(origins = "*")
public class AutomationController {

    @PostMapping("/run")
    public String runSelenium(@RequestBody Map<String, String> request) {
        String url = request.get("url");
        String socketId = request.get("socketId"); // ID from frontend
        System.out.println(url + " " + socketId);
        WebSocketSession session = WebSocketSessionManager.get(socketId);
        if (session == null || !session.isOpen()) {
            return "❌ WebSocket not connected or closed.";
        }

        new Thread(() -> {
            Hello.runSelenium(url, log -> {
                try {
                    session.sendMessage(new TextMessage(log));
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }).start();

        return "✅ Test started";
    }
}

#websocketsession manager 
package com.example.demo;

import org.springframework.web.socket.WebSocketSession;

import java.util.concurrent.ConcurrentHashMap;

public class WebSocketSessionManager {
    private static final ConcurrentHashMap<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

    public static void register(String id, WebSocketSession session) {
        sessions.put(id, session);
    }

    public static WebSocketSession get(String id) {
        return sessions.get(id);
    }

    public static void remove(String id) {
        sessions.remove(id);
    }
}

#websocketconfig.java
package com.example.demo;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new AutomationWebSocketHandler(), "/ws/logs").setAllowedOrigins("*");
    }
}

# Automationwebsockethandler.java
package com.example.demo;

import com.example.demo.WebSocketSessionManager.*;
import org.springframework.web.socket.*;
import org.springframework.web.socket.handler.TextWebSocketHandler;

public class AutomationWebSocketHandler extends TextWebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String query = session.getUri().getQuery(); // e.g. clientId=client-172345789
        String clientId = null;

        if (query != null && query.startsWith("clientId=")) {
            clientId = query.substring("clientId=".length());
        }

        if (clientId != null) {
            System.out.println("🟢 WebSocket connected for clientId: " + clientId);
            WebSocketSessionManager.register(clientId, session);
        } else {
            System.out.println("❌ clientId missing in WebSocket URL");
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        WebSocketSessionManager.remove(session.getId());
        System.out.println("❌ WebSocket closed: " + session.getId());
    }
}


    <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>


