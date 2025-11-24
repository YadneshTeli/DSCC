# Distributed Systems and Cloud Computing (DSCC) Practicals

**Roll No:** C24120

This repository contains practical implementations covering various distributed systems concepts including Socket Programming, Remote Procedure Calls (RPC), Remote Method Invocation (RMI), JDBC with Remote Object Communication, and Distributed Algorithms.

---


## 1. Remote Process Communication

This section demonstrates socket programming in Java using both TCP (Socket) and UDP (Datagram) protocols for client-server communication. 
### 1.1 Multi-Client Chat Server using Socket

**Description:** This practical implements a multi-threaded chat server that can handle multiple clients simultaneously. The server broadcasts messages from one client to all other connected clients, enabling real-time group communication. Uses TCP sockets for reliable message delivery.

**MultiClientChatServer.java:**
```java
import java.io.*;
import java.net.*;
import java.util.*;
 
public class MultiClientChatServer {
    private static Set<ClientHandler> clientHandlers = new HashSet<>();
 
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5000);
            System.out.println("Multi-client Chat Server started...");
 
            while (true) {
                Socket clientSocket = serverSocket.accept();
                System.out.println("New client connected: " + clientSocket.getInetAddress().getHostAddress());
 
                ClientHandler clientHandler = new ClientHandler(clientSocket);
                clientHandlers.add(clientHandler);
                new Thread(clientHandler).start();
            }
 
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
 
    static void broadcast(String message, ClientHandler sender) {
        for (ClientHandler client : clientHandlers) {
            if (client != sender) {
                client.sendMessage(message);
            }
        }
    }
 
    static void removeClient(ClientHandler clientHandler) {
        clientHandlers.remove(clientHandler);
    }
}
 
class ClientHandler implements Runnable {
    private Socket socket;
    private BufferedReader input;
    private PrintWriter output;
    private String clientName;
 
    public ClientHandler(Socket socket) {
        this.socket = socket;
        try {
            input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            output = new PrintWriter(socket.getOutputStream(), true);
            output.println("Enter your name: ");
            clientName = input.readLine();
            broadcast(clientName + " has joined the chat!");
        } catch (IOException e) {
            System.out.println("I/O Error: " + e.getMessage());
        }
    }
 
    @Override
    public void run() {
        String message;
        try {
            while ((message = input.readLine()) != null) {
                if (message.equalsIgnoreCase("exit")) {
                    output.println("You left the chat.");
                    System.out.println(clientName + " disconnected.");
                    broadcast(clientName + " has left the chat.", this);
                    MultiClientChatServer.removeClient(this);
                    break;
                }
                String formattedMessage = clientName + ": " + message;
                System.out.println(formattedMessage);
                MultiClientChatServer.broadcast(formattedMessage, this);
            }
            input.close();
            output.close();
            socket.close();
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
 
    void sendMessage(String message) {
        output.println(message);
    }
 
    void broadcast(String message) {
        MultiClientChatServer.broadcast(message, this);
    }
}

```
**MultiClientChatClient.java:**
```java
import java.io.*;
import java.net.*;
import java.util.Scanner;
 
public class MultiClientChatClient {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket("localhost", 5000);
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
 
            new Thread(() -> {
                String serverMessage;
                try {
                    while ((serverMessage = input.readLine()) != null) {
                        System.out.println(serverMessage);
                    }
                } catch (IOException e) {
                    System.out.println("Connection closed.");
                }
            }).start();
 
             while (true) {
                String message = scanner.nextLine();
                output.println(message);
                if (message.equalsIgnoreCase("exit")) {
                    break;
                }
            }
 
            scanner.close();
            input.close();
            output.close();
            socket.close();
 
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

```
**Output:**

---

### 1.2 Server Calculator using RPC (UDP Datagram)

**Description:** This practical demonstrates a calculator server using UDP (Datagram) sockets implementing basic RPC concepts. The server receives arithmetic operation requests from clients (ADD, SUB, MUL, DIV) and returns the computed results. UDP provides connectionless, fast communication suitable for simple request-response patterns.

**CalculatorServer.java:**
```java
package socket_prac;
import java.net.*;
import java.io.*;
public class CalculatorServer {
    public static void main(String[] args) {
        try {
            DatagramSocket serverSocket = new DatagramSocket(5000);
            System.out.println("UDP Calculator Server started. Waiting for client requests...");
            byte[] receiveBuffer = new byte[1024];
            byte[] sendBuffer;
            while (true) {
                DatagramPacket receivePacket = new DatagramPacket(receiveBuffer, receiveBuffer.length);
                serverSocket.receive(receivePacket);
                String clientRequest = new String(receivePacket.getData(), 0, receivePacket.getLength());
                System.out.println("Received from client: " + clientRequest);

                if (clientRequest.equalsIgnoreCase("exit")) {
                    System.out.println("Client disconnected.");
                    break;
                }
                String[] parts = clientRequest.split(" ");
                String response;

                if (parts.length != 3) {
                    response = "Invalid format! Use: OPERATION num1 num2";
                } else {
                    try {
                        String operation = parts[0].toUpperCase();
                        double num1 = Double.parseDouble(parts[1]);
                        double num2 = Double.parseDouble(parts[2]);
                        double result;
                        switch (operation) {
                            case "ADD":
                                result = num1 + num2;
                                response = "Result: " + result;
                                break;
                            case "SUB":
                                result = num1 - num2;
                                response = "Result: " + result;
                                break;
                            case "MUL":
                                result = num1 * num2;
                                response = "Result: " + result;
                                break;
                            case "DIV":
                                if (num2 == 0) {
                                    response = "Error: Division by zero!";
                                } else {
                                    result = num1 / num2;
                                    response = "Result: " + result;
                                }
                                break;
                            default:
                                response = "Unknown operation. Use ADD, SUB, MUL, or DIV.";
                                break;
                        }
                    } catch (Exception e) {
                        response = "Error: Invalid numbers!";
                    }
                }
                InetAddress clientAddress = receivePacket.getAddress();
                int clientPort = receivePacket.getPort();
                sendBuffer = response.getBytes();
                DatagramPacket sendPacket = new DatagramPacket(sendBuffer, sendBuffer.length, clientAddress, clientPort);
                serverSocket.send(sendPacket);
            }
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

```
**CalculatorClient.java:**
```java
package socket_prac;
import java.net.*;
import java.io.*;
import java.util.Scanner;
public class CalculatorClient {
    public static void main(String[] args) {
        try {
            DatagramSocket socket = new DatagramSocket();
            InetAddress serverAddress = InetAddress.getByName("localhost");
            Scanner scanner = new Scanner(System.in);
            System.out.println("Connected to UDP Calculator Server.");
            System.out.println("\nAvailable operations: ADD, SUB, MUL, DIV");
            System.out.println("Format: OPERATION num1 num2 (e.g., ADD 5 10)");
            System.out.println("Type 'exit' to quit.\n");
            byte[] sendBuffer;
            byte[] receiveBuffer = new byte[1024];
            while (true) {
                System.out.print("Enter command: ");
                String userInput = scanner.nextLine();
                sendBuffer = userInput.getBytes();
                DatagramPacket sendPacket = new DatagramPacket(sendBuffer, sendBuffer.length, serverAddress, 5000);
                socket.send(sendPacket);
                if (userInput.equalsIgnoreCase("exit")) {
                    System.out.println("Closing connection...");
                    break;
                }
                DatagramPacket receivePacket = new DatagramPacket(receiveBuffer, receiveBuffer.length);
                socket.receive(receivePacket);
                String response = new String(receivePacket.getData(), 0, receivePacket.getLength());
                System.out.println("Server: " + response);
            }

            scanner.close();
            socket.close();
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

```
**Output:**

---

### 1.3 Date Time Server using RPC (UDP Datagram)

**Description:** This practical implements a Date-Time server using UDP datagrams. Clients can request either the current date or time from the server. The server processes the request and sends back the formatted date/time information. Demonstrates simple RPC implementation with stateless communication.

**DateTimeServer.java:**
```java
package socket_prac;
import java.net.*;
import java.io.*;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
public class DateTimeServer {
    public static void main(String[] args) {
        try {
            DatagramSocket serverSocket = new DatagramSocket(5001);
            System.out.println("UDP Date-Time Server started...");
            System.out.println("Waiting for client requests...");
            byte[] receiveBuffer = new byte[1024];
            byte[] sendBuffer;
            while (true) {
                DatagramPacket receivePacket = new DatagramPacket(receiveBuffer, receiveBuffer.length);
                serverSocket.receive(receivePacket);

                String clientRequest = new String(receivePacket.getData(), 0, receivePacket.getLength());
                System.out.println("Received from client: " + clientRequest);

                if (clientRequest.equalsIgnoreCase("exit")) {
                    System.out.println("Client disconnected.");
                    break;
                }
                String response;
                switch (clientRequest.toLowerCase()) {
                    case "date":
                        LocalDate date = LocalDate.now();
                        response = "Current Date: " + date.format(DateTimeFormatter.ofPattern("dd-MM-yyyy"));
                        break;
                    case "time":
                        LocalTime time = LocalTime.now();
                        response = "Current Time: " + time.format(DateTimeFormatter.ofPattern("HH:mm:ss"));
                        break;
                    default:
                        response = "Invalid command! Use 'date' or 'time'.";
                        break;
                }

                InetAddress clientAddress = receivePacket.getAddress();
                int clientPort = receivePacket.getPort();
                sendBuffer = response.getBytes();
                DatagramPacket sendPacket = new DatagramPacket(sendBuffer, sendBuffer.length, clientAddress, clientPort);
                serverSocket.send(sendPacket);
            }
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

```
**DateTimeClient.java:**
```java
package socket_prac;

import java.net.*;
import java.io.*;
import java.util.Scanner;

public class DateTimeClient {
    public static void main(String[] args) {
        try {
            DatagramSocket socket = new DatagramSocket();
            InetAddress serverAddress = InetAddress.getByName("localhost");
            Scanner scanner = new Scanner(System.in);

            System.out.println("Connected to UDP Date-Time Server.");
            System.out.println("\nAvailable commands: 'date', 'time'");
            System.out.println("Type 'exit' to quit.\n");

            byte[] sendBuffer;
            byte[] receiveBuffer = new byte[1024];

            while (true) {
                System.out.print("Enter command: ");
                String userInput = scanner.nextLine();
                sendBuffer = userInput.getBytes();

                DatagramPacket sendPacket = new DatagramPacket(sendBuffer, sendBuffer.length, serverAddress, 5001);
                socket.send(sendPacket);

                if (userInput.equalsIgnoreCase("exit")) {
                    System.out.println("Closing connection...");
                    break;
                }

                DatagramPacket receivePacket = new DatagramPacket(receiveBuffer, receiveBuffer.length);
                socket.receive(receivePacket);
                String response = new String(receivePacket.getData(), 0, receivePacket.getLength());
                System.out.println("Server: " + response);
            }

            scanner.close();
            socket.close();
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}


```
**Output:**

---

### 1.4 Server Calculator using RPC (TCP Server Socket)

**Description:** This practical implements a calculator server using TCP sockets (ServerSocket) for reliable connection-oriented communication. Unlike the UDP version, this maintains a persistent connection with the client, allowing multiple calculations in a single session. Provides better reliability for critical arithmetic operations.

**CalculatorServer.java:**
```java
package socket_prac;
import java.io.*;
import java.net.*;
public class CalculatorServer {
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5000);
            System.out.println("Server started. Waiting for client connection...");
            Socket socket = serverSocket.accept();
            System.out.println("Client connected!");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            String clientRequest;
            while ((clientRequest = input.readLine()) != null) {
                if (clientRequest.equalsIgnoreCase("exit")) {
                    System.out.println("Client disconnected.");
                    break;
                }
                System.out.println("Received from client: " + clientRequest);
                String[] parts = clientRequest.split(" ");
                if (parts.length != 3) {
                    output.println("Invalid format! Use: OPERATION num1 num2");
                    continue;
                }
                String operation = parts[0].toUpperCase();
                double num1 = Double.parseDouble(parts[1]);
                double num2 = Double.parseDouble(parts[2]);
                double result = 0.0;
                switch (operation) {
                    case "ADD":
                        result = num1 + num2;
                        break;
                    case "SUB":
                        result = num1 - num2;
                        break;
                    case "MUL":
                        result = num1 * num2;
                        break;
                    case "DIV":
                        if (num2 == 0) {
                            output.println("Error: Division by zero!");
                            continue;
                        }
                        result = num1 / num2;
                        break;
                    default:
                        output.println("Unknown operation. Use ADD, SUB, MUL, or DIV.");
                        continue;
                }
                output.println("Result: " + result);
            }
            input.close();
            output.close();
            socket.close();
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}
CalculatorClient.java:
package socket_prac;
import java.io.*;
import java.net.*;
import java.util.*;
public class CalculatorClient {
    public static void main(String[] args) {
        try {
           Socket socket = new Socket("localhost", 5000);
            System.out.println("Connected to Calculator Server.");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
            System.out.println("\nAvailable operations: ADD, SUB, MUL, DIV");
            System.out.println("Format: OPERATION num1 num2 (e.g., ADD 5 10)");
            System.out.println("Type 'exit' to quit.\n");
            String userInput;
            while (true) {
                System.out.print("Enter command: ");
                userInput = scanner.nextLine();
                output.println(userInput);
                if (userInput.equalsIgnoreCase("exit")) {
                    System.out.println("Closing connection...");
                    break;
                }
                String response = input.readLine();
                System.out.println("Server: " + response);
            }
            scanner.close();
            input.close();
            output.close();
            socket.close();
 
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}
 
 
 
 
 
 
Output:




```
**Output:**

---

### 1.5 Date Time Server using RPC (TCP Server Socket)

**Description:** This practical implements a Date-Time server using TCP sockets, providing reliable connection-oriented communication. Clients maintain a persistent connection and can request date or time information multiple times. The TCP connection ensures reliable delivery of date/time data.

**DateTimeServer.java:**
```java
package socket_prac;
import java.io.*;
import java.net.*;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
public class DateTimeServer {
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5001);
            System.out.println("Date-Time Server started...");
            System.out.println("Waiting for client connection...");
            Socket socket = serverSocket.accept();
            System.out.println("Client connected!");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            String clientRequest;
            while ((clientRequest = input.readLine()) != null) {
                System.out.println("Received from client: " + clientRequest);
                if (clientRequest.equalsIgnoreCase("exit")) {
                    output.println("Connection closed. Goodbye!");
                    System.out.println("Client disconnected.");
                    break;
                }
                switch (clientRequest.toLowerCase()) {
                    case "date":
                        LocalDate date = LocalDate.now();
                        output.println("Current Date: " + date.format(DateTimeFormatter.ofPattern("dd-MM-yyyy")));
                        break;
                    case "time":
                        LocalTime time = LocalTime.now();
                        output.println("Current Time: " + time.format(DateTimeFormatter.ofPattern("HH:mm:ss")));
                        break;
                    default:
                        output.println("Invalid command! Use 'date' or 'time'.");
                        break;
                }
            }
            input.close();
            output.close();
            socket.close();
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 

DateTimeClient.java:
package socket_prac;
 
import java.io.*;
import java.net.*;
import java.util.Scanner;
 
public class DateTimeClient {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket("localhost", 5001);
            System.out.println("Connected to Date-Time Server.");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
            System.out.println("\nAvailable commands: 'date', 'time'");
            System.out.println("Type 'exit' to quit.\n");
            String userInput;
            while (true) {
                System.out.print("Enter command: ");
                userInput = scanner.nextLine();
                output.println(userInput);
                if (userInput.equalsIgnoreCase("exit")) {
                    System.out.println("Closing connection...");
                    break;
                }
                String response = input.readLine();
                System.out.println("Server: " + response);
            }
            scanner.close();
            input.close();
            output.close();
            socket.close();
 
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

 


Output:
 
  



```
**Output:**

---

## 2. Remote Procedure Call

This section covers server-client applications implementing RPC concepts for various computational tasks using TCP sockets.

### 2.1 Calculator Server with Multiple Operations

**Description:** A comprehensive calculator server implementing multiple arithmetic operations (ADD, SUB, MUL, DIV). The server processes operation requests from clients and returns computed results. Demonstrates basic RPC pattern with procedure-based remote execution.

**CalculatorServer.java:**
```java
package socket_prac;
import java.io.*;
import java.net.*;
public class CalculatorServer {
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5000);
            System.out.println("Server started. Waiting for client connection...");
            Socket socket = serverSocket.accept();
            System.out.println("Client connected!");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            String clientRequest;
            while ((clientRequest = input.readLine()) != null) {
                if (clientRequest.equalsIgnoreCase("exit")) {
                    System.out.println("Client disconnected.");
                    break;
                }
                System.out.println("Received from client: " + clientRequest);
                String[] parts = clientRequest.split(" ");
                if (parts.length != 3) {
                    output.println("Invalid format! Use: OPERATION num1 num2");
                    continue;
                }
                String operation = parts[0].toUpperCase();
                double num1 = Double.parseDouble(parts[1]);
                double num2 = Double.parseDouble(parts[2]);
                double result = 0.0;
                switch (operation) {
                    case "ADD":
                        result = num1 + num2;
                        break;
                    case "SUB":
                        result = num1 - num2;
                        break;
                    case "MUL":
                        result = num1 * num2;
                        break;
                    case "DIV":
                        if (num2 == 0) {
                            output.println("Error: Division by zero!");
                            continue;
                        }
                        result = num1 / num2;
                        break;
                    default:
                        output.println("Unknown operation. Use ADD, SUB, MUL, or DIV.");
                        continue;
                }
                output.println("Result: " + result);
            }
            input.close();
            output.close();
            socket.close();
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}
CalculatorClient.java:
package socket_prac;
import java.io.*;
import java.net.*;
import java.util.*;
public class CalculatorClient {
    public static void main(String[] args) {
        try {
           Socket socket = new Socket("localhost", 5000);
            System.out.println("Connected to Calculator Server.");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
            System.out.println("\nAvailable operations: ADD, SUB, MUL, DIV");
            System.out.println("Format: OPERATION num1 num2 (e.g., ADD 5 10)");
            System.out.println("Type 'exit' to quit.\n");
            String userInput;
            while (true) {
                System.out.print("Enter command: ");
                userInput = scanner.nextLine();
                output.println(userInput);
                if (userInput.equalsIgnoreCase("exit")) {
                    System.out.println("Closing connection...");
                    break;
                }
                String response = input.readLine();
                System.out.println("Server: " + response);
            }
            scanner.close();
            input.close();
            output.close();
            socket.close();
 
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}
 
 
 
 
 
 
Output:




```
**Output:**

---

### 2.2 Date Time Server with date() and time() Methods

**Description:** This practical implements a Date-Time server with specific remote procedures for retrieving date and time separately. Clients can invoke date() or time() methods remotely, demonstrating procedural remote invocation pattern.

**DateTimeServer.java:**
```java
package socket_prac;
import java.io.*;
import java.net.*;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
public class DateTimeServer {
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5001);
            System.out.println("Date-Time Server started...");
            System.out.println("Waiting for client connection...");
            Socket socket = serverSocket.accept();
            System.out.println("Client connected!");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            String clientRequest;
            while ((clientRequest = input.readLine()) != null) {
                System.out.println("Received from client: " + clientRequest);
                if (clientRequest.equalsIgnoreCase("exit")) {
                    output.println("Connection closed. Goodbye!");
                    System.out.println("Client disconnected.");
                    break;
                }
                switch (clientRequest.toLowerCase()) {
                    case "date":
                        LocalDate date = LocalDate.now();
                        output.println("Current Date: " + date.format(DateTimeFormatter.ofPattern("dd-MM-yyyy")));
                        break;
                    case "time":
                        LocalTime time = LocalTime.now();
                        output.println("Current Time: " + time.format(DateTimeFormatter.ofPattern("HH:mm:ss")));
                        break;
                    default:
                        output.println("Invalid command! Use 'date' or 'time'.");
                        break;
                }
            }
            input.close();
            output.close();
            socket.close();
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 

DateTimeClient.java:
package socket_prac;
 
import java.io.*;
import java.net.*;
import java.util.Scanner;
 
public class DateTimeClient {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket("localhost", 5001);
            System.out.println("Connected to Date-Time Server.");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
            System.out.println("\nAvailable commands: 'date', 'time'");
            System.out.println("Type 'exit' to quit.\n");
            String userInput;
            while (true) {
                System.out.print("Enter command: ");
                userInput = scanner.nextLine();
                output.println(userInput);
                if (userInput.equalsIgnoreCase("exit")) {
                    System.out.println("Closing connection...");
                    break;
                }
                String response = input.readLine();
                System.out.println("Server: " + response);
            }
            scanner.close();
            input.close();
            output.close();
            socket.close();
 
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

 


```
**Output:**

---

### 2.3 Time Addition Server

**Description:** This practical demonstrates a server that performs time arithmetic. Clients send two time values (hours and minutes), and the server adds them properly handling minute-to-hour conversion. The result is displayed both in hours:minutes format and total minutes format. Shows complex data processing in RPC.

**Requirements:**
1. User enters two time values from client (e.g., 3 hours 40 minutes and 2 hours 50 minutes)
2. Server adds hour and minute components separately with proper conversion
3. Client receives and displays results in both formats:
   - In hours and minutes (HH:MM)
   - Total minutes only

**TimeAdditionServer.java**
package socket_prac;
import java.io.*;
import java.net.*;
public class TimeAdditionServer {
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5002);
            System.out.println("Time Addition Server started...");
            System.out.println("Waiting for client connection...");
            Socket socket = serverSocket.accept();
            System.out.println("Client connected!");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            String data = input.readLine();
            System.out.println("Received from client: " + data);
            String[] parts = data.split(" ");
            if (parts.length != 4) {
                output.println("Invalid input! Send data as: h1 m1 h2 m2");
            } else {
                int h1 = Integer.parseInt(parts[0]);
                int m1 = Integer.parseInt(parts[1]);
                int h2 = Integer.parseInt(parts[2]);
                int m2 = Integer.parseInt(parts[3]);
                int totalMinutes = m1 + m2;
                int totalHours = h1 + h2;
                totalHours += totalMinutes / 60;
                totalMinutes = totalMinutes % 60;
                int totalInMinutes = totalHours * 60 + totalMinutes;
                String resultInHours = String.format("%02d:%02d Hours", totalHours, totalMinutes);
                String resultInMinutes = totalInMinutes + " Minutes";
               output.println("Result in Hours: " + resultInHours);
                output.println("Result in Minutes: " + resultInMinutes);
            }
            input.close();
            output.close();
            socket.close();
            serverSocket.close();
            System.out.println("Server closed.");
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

**TimeAdditionClient.java:**
```java
package socket_prac;
 
import java.io.*;
import java.net.*;
import java.util.Scanner;
 
public class TimeAdditionClient {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket("localhost", 5002);
            System.out.println("Connected to Time Addition Server.");
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
            System.out.println("Enter first time:");
            System.out.print("Hours: ");
            int h1 = scanner.nextInt();
            System.out.print("Minutes: ");
            int m1 = scanner.nextInt();
            System.out.println("Enter second time:");
            System.out.print("Hours: ");
            int h2 = scanner.nextInt();
            System.out.print("Minutes: ");
            int m2 = scanner.nextInt();
            String data = h1 + " " + m1 + " " + h2 + " " + m2;
            output.println(data);
            String response1 = input.readLine();
            String response2 = input.readLine();
            System.out.println("\n" + response1);
            System.out.println(response2);
            scanner.close();
            input.close();
            output.close();
            socket.close();
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

```
**Output:**

---

### 2.4 String Reverse and Palindrome Checker

**Description:** This practical implements a distributed palindrome checker. The client sends a string to the server, which reverses it and sends it back. The client then compares the original and reversed strings to determine if it's a palindrome. Demonstrates distributed string processing and client-side decision making.

**Workflow:**
1. Client sends a string to the server
2. Server reverses the string and returns it
3. Client receives reversed string and checks for palindrome

**ReverseServer.java:**
```java
import java.io.*;
import java.net.*;
 
public class ReverseServer {
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5000);
            System.out.println("Reverse Server started...");
            Socket socket = serverSocket.accept();
            System.out.println("Client connected!");
 
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
 
            String str = input.readLine();
            String reversed = new StringBuilder(str).reverse().toString();
            output.println(reversed);
 
            input.close();
            output.close();
            socket.close();
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 


ReverseClient.java:
import java.io.*;
import java.net.*;
import java.util.Scanner;
 
public class ReverseClient {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket("localhost", 5000);
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
 
            System.out.print("Enter a string: ");
            String userInput = scanner.nextLine();
            output.println(userInput);
 
            String reversed = input.readLine();
            System.out.println("Reversed string from server: " + reversed);
 
            if (userInput.equalsIgnoreCase(reversed))
                System.out.println("Result: The string is a Palindrome!");
            else
                System.out.println("Result: The string is NOT a Palindrome!");
 
            scanner.close();
            input.close();
            output.close();
            socket.close();
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

```
**Output:**

---

### 2.5 Odd/Even Server with Multiplication Table

**Description:** This practical demonstrates conditional logic execution in distributed systems. The server determines if a number is odd or even, and based on the result, the client generates a multiplication table (only for even numbers). Shows server-side decision making with client-side conditional processing.

**Workflow:**
1. Client sends a number to the server
2. Server determines if the number is odd or even
3. Client receives the result
4. If even, client calculates and displays the multiplication table (1-10)

**OddEvenServer.java:**
```java
import java.io.*;
import java.net.*;
 
public class OddEvenServer {
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(5000);
            System.out.println("Odd-Even Server started...");
            Socket socket = serverSocket.accept();
            System.out.println("Client connected.");
 
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
 
            String numStr = input.readLine();
            int num = Integer.parseInt(numStr);
            String result = (num % 2 == 0) ? "even" : "odd";
            output.println(result);
 
            input.close();
            output.close();
            socket.close();
            serverSocket.close();
        } catch (IOException e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 



OddEvenClient.java:
import java.io.*;
import java.net.*;
import java.util.Scanner;
 
public class OddEvenClient {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket("localhost", 5000);
            BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter output = new PrintWriter(socket.getOutputStream(), true);
            Scanner scanner = new Scanner(System.in);
 
            System.out.print("Enter a number: ");
            int num = scanner.nextInt();
            output.println(num);
 
            String response = input.readLine();
            System.out.println("Server says: The number is " + response);
 
            if (response.equalsIgnoreCase("even")) {
                System.out.println("Multiplication Table of " + num + ":");
                for (int i = 1; i <= 10; i++)
                    System.out.println(num + " x " + i + " = " + (num * i));
            }
 
            scanner.close();
            input.close();
            output.close();
            socket.close();
        } catch (IOException e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

```
**Output:**

---

## 3. RMI Based Exercises

This section covers Remote Method Invocation (RMI) implementations for various computational tasks. RMI provides true object-oriented distributed computing where clients can invoke methods on remote objects.

### 3.1 Addition Service using RMI

**Description:** This practical implements basic RMI architecture with a simple addition service. Demonstrates the core components of RMI: Remote Interface, Remote Object Implementation, RMI Server (with registry), and RMI Client. Shows how to perform remote method invocation for arithmetic operations.

**AdditionInterface.java (Remote Interface):**
```java
import java.rmi.Remote;
import java.rmi.RemoteException;
 
public interface AdditionInterface extends Remote {
    int add(int a, int b) throws RemoteException;
}

```
**AdditionImpl.java (Remote Object Implementation):**
```java
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
 
public class AdditionImpl extends UnicastRemoteObject implements AdditionInterface {
 
    protected AdditionImpl() throws RemoteException {
        super();
    }
 
    @Override
    public int add(int a, int b) throws RemoteException {
        return a + b;
    }
}

```
**AdditionServer.java (RMI Server):**
```java
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
 
public class AdditionServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099); // Start RMI registry
            AdditionImpl addition = new AdditionImpl();
            Naming.rebind("rmi://localhost/AdditionService", addition);
            System.out.println("Addition RMI Server is ready...");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 


AdditionClient.java (RMI Client):
import java.rmi.Naming;
import java.util.Scanner;
 
public class AdditionClient {
    public static void main(String[] args) {
        try {
            AdditionInterface addition = (AdditionInterface) Naming.lookup("rmi://localhost/AdditionService");
            Scanner scanner = new Scanner(System.in);
 
            System.out.print("Enter first number: ");
            int a = scanner.nextInt();
            System.out.print("Enter second number: ");
            int b = scanner.nextInt();
 
            int result = addition.add(a, b);
            System.out.println("Result from server: " + result);
 
            scanner.close();
        } catch (Exception e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

```
**Output:**

---

### 3.2 Date Time Service using RMI

**Description:** This practical implements a time service using RMI that retrieves current date and time from the server. The server method returns formatted date-time string to the client. Demonstrates RMI with return of complex data types (String) and server-side time processing.

**TimeService.java (Remote Interface):**
```java
import java.rmi.Remote;
import java.rmi.RemoteException;
 
public interface TimeService extends Remote {
    String getDateTime() throws RemoteException;
}

```
**TimeServiceImpl.java (Remote Object Implementation):**
```java
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
 
public class TimeServiceImpl extends UnicastRemoteObject implements TimeService {
 
    protected TimeServiceImpl() throws RemoteException {
        super();
    }
 
    @Override
    public String getDateTime() throws RemoteException {
        LocalDateTime now = LocalDateTime.now();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        return now.format(formatter);
    }
}

```
**TimeServer.java (RMI Server):**
```java
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
 
public class TimeServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099); // Start RMI registry
            TimeServiceImpl timeService = new TimeServiceImpl();
            Naming.rebind("rmi://localhost/TimeService", timeService);
            System.out.println("Time RMI Server is ready...");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 


TimeClient.java (RMI Client):
import java.rmi.Naming;
 
public class TimeClient {
    public static void main(String[] args) {
        try {
            TimeService timeService = (TimeService) Naming.lookup("rmi://localhost/TimeService");
            String serverDateTime = timeService.getDateTime();
            System.out.println("Server Date and Time: " + serverDateTime);
        } catch (Exception e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}
Output:
 




```
**Output:**

---

### 3.3 Equation Solver using RMI - (a+b)²

**Description:** This practical implements an algebraic equation solver using RMI. The server computes the expansion of (a+b)² = a² + 2ab + b² given values of a and b. Demonstrates RMI with mathematical computations and parameter passing.

**EquationService.java (Remote Interface):**
```java
package RMI_based;
import java.rmi.Remote;
import java.rmi.RemoteException;
 
public interface EquationService extends Remote {
    int solveEquation(int a, int b) throws RemoteException;
}
 
EquationServiceImpl.java (Remote Object Implementation):
package RMI_based;
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
 
public class EquationServiceImpl extends UnicastRemoteObject implements EquationService {
 
    protected EquationServiceImpl() throws RemoteException {
        super();
    }
 
    @Override
    public int solveEquation(int a, int b) throws RemoteException {
        // Using formula (a+b)^2 = a^2 + 2ab + b^2
        return (a * a) + (2 * a * b) + (b * b);
    }
}

```
**EquationServer.java (RMI Server):**
```java
package RMI_based;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
 
public class EquationServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099); // Start RMI registry
            EquationServiceImpl equationService = new EquationServiceImpl();
            Naming.rebind("rmi://localhost/EquationService", equationService);
            System.out.println("Equation RMI Server is ready...");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 


EquationClient.java (RMI Client):
package RMI_based;
import java.rmi.Naming;
import java.util.Scanner;
 
public class EquationClient {
    public static void main(String[] args) {
        try {
            EquationService equationService = (EquationService) Naming.lookup("rmi://localhost/EquationService");
            Scanner scanner = new Scanner(System.in);
 
            System.out.print("Enter value of a: ");
            int a = scanner.nextInt();
            System.out.print("Enter value of b: ");
            int b = scanner.nextInt();
 
            int result = equationService.solveEquation(a, b);
            System.out.println("Result of (a+b)^2 = a^2 + 2ab + b^2 is: " + result);
 
            scanner.close();
        } catch (Exception e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}

```
**Output:**

---

### 3.4 Dual Equation Solver using RMI - (a+b)² and (a-b)²

**Description:** This practical extends the equation solver to compute both (a+b)² and (a-b)² algebraic expansions. The server returns an array of results to the client. Demonstrates RMI with multiple computations and array return types.

**EquationService.java (Remote Interface):**
```java
package RMI_based;
import java.rmi.Remote;
import java.rmi.RemoteException;
 
public interface EquationService extends Remote {
    int solveEquation(int a, int b) throws RemoteException;
}
 
EquationServiceImpl.java (Remote Object Implementation):
package RMI_based;
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
 
public class EquationServiceImpl extends UnicastRemoteObject implements EquationService {
 
    protected EquationServiceImpl() throws RemoteException {
        super();
    }
 
    @Override
    public int[] solveEquations(int a, int b) throws RemoteException {
        int sumSquare = (a * a) + (2 * a * b) + (b * b);  // (a+b)^2
        int diffSquare = (a * a) - (2 * a * b) + (b * b); // (a-b)^2
        return new int[]{sumSquare, diffSquare};
    }
}

```
**EquationServer.java (RMI Server):**
```java
package RMI_based;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
 
public class EquationServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099); // Start RMI registry
            EquationServiceImpl equationService = new EquationServiceImpl();
            Naming.rebind("rmi://localhost/EquationService", equationService);
            System.out.println("Equation RMI Server is ready...");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

```
**EquationClient.java (RMI Client):**
```java
package RMI_based;
import java.rmi.Naming;
import java.util.Scanner;
 
public class EquationClient {
    public static void main(String[] args) {
        try {
            EquationService equationService = (EquationService) Naming.lookup("rmi://localhost/EquationService");
            Scanner scanner = new Scanner(System.in);
 
            System.out.print("Enter value of a: ");
            int a = scanner.nextInt();
            System.out.print("Enter value of b: ");
            int b = scanner.nextInt();
 
            int[] results = equationService.solveEquations(a, b);
            System.out.println("(a+b)^2 = " + results[0]);
            System.out.println("(a-b)^2 = " + results[1]);
 
            scanner.close();
        } catch (Exception e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}
Output:
 

 


 


```
## 4. Remote Method Invocation with Graphical User Interface

This section combines RMI with Java Swing GUI, creating user-friendly interfaces for distributed applications. Demonstrates integration of RMI backend services with interactive frontend clients.

### 4.1 Addition GUI using RMI

**Description:** This practical implements a graphical calculator for addition using RMI. The GUI client provides text fields for input numbers and displays results. The server performs the addition remotely. Shows integration of Swing components with RMI for user-friendly distributed computing.

**AdditionService.java (Remote Interface):
package RMI_GUI;
import java.rmi.Remote;
import java.rmi.RemoteException;
 
public interface AdditionService extends Remote {
    int add(int a, int b) throws RemoteException;
}

**AdditionServiceImpl.java (Remote Object Implementation):**
```java
package RMI_GUI;
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
 
public class AdditionServiceImpl extends UnicastRemoteObject implements AdditionService {
 
    protected AdditionServiceImpl() throws RemoteException {
        super();
    }
 
    @Override
    public int add(int a, int b) throws RemoteException {
        return a + b;
    }
}

```
**AdditionServer.java (RMI Server):**
```java
package RMI_GUI;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
public class AdditionServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099);
            AdditionServiceImpl service = new AdditionServiceImpl();
            Naming.rebind("rmi://localhost/AdditionService", service);
            System.out.println("Addition RMI Server ready");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}
 

 
 
 
AdditionClientGUI.java (RMI Client):
package RMI_GUI;
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.rmi.Naming;
 
public class AdditionClientGUI extends JFrame {
 
    private JTextField num1Field;
    private JTextField num2Field;
    private JButton addButton;
    private JLabel resultLabel;
 
    public AdditionClientGUI() {
        setTitle("RMI Addition Client");
        setSize(400, 200);
        setLayout(new GridLayout(4, 2, 10, 10));
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
 
        num1Field = new JTextField();
        num2Field = new JTextField();
        addButton = new JButton("Add");
        resultLabel = new JLabel("Result: ");
 
        add(new JLabel("Enter first number: "));
        add(num1Field);
        add(new JLabel("Enter second number: "));
        add(num2Field);
        add(addButton);
        add(resultLabel);
 
        addButton.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                try {
                    int a = Integer.parseInt(num1Field.getText());
                    int b = Integer.parseInt(num2Field.getText());
                    AdditionService service = (AdditionService) Naming.lookup("rmi://localhost/AdditionService");
                    int result = service.add(a, b);
                    resultLabel.setText("Result: " + result);
                } catch (Exception ex) {
                    resultLabel.setText("Error: " + ex.getMessage());
                }
            }
        });
 
        setVisible(true);
    }
 
    public static void main(String[] args) {
        new AdditionClientGUI();
    }
}
 
Output:
 

 


 


```
### 4.2 Factorial Calculator GUI using RMI

**Description:** This practical implements a factorial calculator with GUI using RMI. The client provides an input field for the number, and the server computes the factorial remotely. Demonstrates RMI with iterative algorithms and larger result values (using long data type).

**FactorialService.java (Remote Interface):
package RMI_GUI;
import java.rmi.Remote;
import java.rmi.RemoteException;
 
public interface FactorialService extends Remote {
    long findFactorial(int n) throws RemoteException;
}

**FactorialServiceImpl.java (Remote Object Implementation):**
```java
package RMI_GUI;
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
 
public class FactorialServiceImpl extends UnicastRemoteObject implements FactorialService {
 
    protected FactorialServiceImpl() throws RemoteException {
        super();
    }
 
    @Override
    public long findFactorial(int n) throws RemoteException {
        long fact = 1;
        for (int i = 1; i <= n; i++)
            fact *= i;
        return fact;
    }
}

```
**FactorialServer.java (RMI Server):**
```java
package RMI_GUI;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
 
public class FactorialServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099);
            FactorialServiceImpl service = new FactorialServiceImpl();
            Naming.rebind("rmi://localhost/FactorialService", service);
            System.out.println("Factorial RMI Server ready");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}
 

 
FactorialClientGUI.java (RMI Client):
package RMI_GUI;
 
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.rmi.Naming;
 
public class FactorialClientGUI extends JFrame {
 
    private JTextField inputField;
    private JButton calcButton;
    private JLabel resultLabel;
 
    public FactorialClientGUI() {
        setTitle("RMI Factorial Calculator");
        setSize(400, 200);
        setLayout(new GridLayout(3, 2, 10, 10));
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
 
        inputField = new JTextField();
        calcButton = new JButton("Find Factorial");
        resultLabel = new JLabel("Result: ");
 
        add(new JLabel("Enter a number: "));
        add(inputField);
        add(calcButton);
        add(resultLabel);
 
        calcButton.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                try {
                    int num = Integer.parseInt(inputField.getText());
                    FactorialService service = (FactorialService) Naming.lookup("rmi://localhost/FactorialService");
                    long fact = service.findFactorial(num);
                    resultLabel.setText("Result: " + fact);
                } catch (Exception ex) {
                    resultLabel.setText("Error: " + ex.getMessage());
                }
            }
        });
 
        setVisible(true);
    }
 
    public static void main(String[] args) {
        new FactorialClientGUI();
    }
}
 
 
 
 
 
 
Output:
 

 




```
### 4.3 Basic Calculator GUI using RMI

**Description:** This practical implements a complete basic calculator with GUI supporting all four arithmetic operations (add, subtract, multiply, divide) using RMI. The server provides separate methods for each operation with error handling (division by zero). Demonstrates comprehensive RMI service with multiple methods.

**CalculatorService.java (Remote Interface):
import java.rmi.Remote;
import java.rmi.RemoteException;
 public interface CalculatorService extends Remote {
    double add(double a, double b) throws RemoteException;
    double subtract(double a, double b) throws RemoteException;
    double multiply(double a, double b) throws RemoteException;
    double divide(double a, double b) throws RemoteException;
}

**CalculatorServiceImpl.java (Remote Object Implementation):**
```java
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
public class CalculatorServiceImpl extends UnicastRemoteObject implements CalculatorService {
    protected CalculatorServiceImpl() throws RemoteException {
        super();
    }
    public double add(double a, double b) throws RemoteException {
        return a + b;
    }
    public double subtract(double a, double b) throws RemoteException {
        return a - b;
    }
    public double multiply(double a, double b) throws RemoteException {
        return a * b;
    }
    public double divide(double a, double b) throws RemoteException {
        if (b == 0) throw new RemoteException("Division by zero is not allowed");
        return a / b;
    }
}

```
**CalculatorServer.java (RMI Server):**
```java
package RMI_GUI;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
public class AdditionServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099);
            AdditionServiceImpl service = new AdditionServiceImpl();
            Naming.rebind("rmi://localhost/AdditionService", service);
            System.out.println("Addition RMI Server ready");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

```
**CalculatorClientGUI.java (RMI Client with GUI):**
```java
package RMI_GUI;
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.rmi.Naming;
 
public class AdditionClientGUI extends JFrame {
 
    private JTextField num1Field;
    private JTextField num2Field;
    private JButton addButton;
    private JLabel resultLabel;
 
    public AdditionClientGUI() {
        setTitle("RMI Addition Client");
        setSize(400, 200);
        setLayout(new GridLayout(4, 2, 10, 10));
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
 
        num1Field = new JTextField();
        num2Field = new JTextField();
        addButton = new JButton("Add");
        resultLabel = new JLabel("Result: ");
 
        add(new JLabel("Enter first number: "));
        add(num1Field);
        add(new JLabel("Enter second number: "));
        add(num2Field);
        add(addButton);
        add(resultLabel);
 
        addButton.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                try {
                    int a = Integer.parseInt(num1Field.getText());
                    int b = Integer.parseInt(num2Field.getText());
                    AdditionService service = (AdditionService) Naming.lookup("rmi://localhost/AdditionService");
                    int result = service.add(a, b);
                    resultLabel.setText("Result: " + result);
                } catch (Exception ex) {
                    resultLabel.setText("Error: " + ex.getMessage());
                }
            }
        });
 
        setVisible(true);
    }
 
    public static void main(String[] args) {
        new AdditionClientGUI();
    }
}
 
Output:
 

 


 



```
### 4.4 Greatest Number Finder GUI using RMI

**Description:** This practical implements a comparison service using RMI with GUI. The client inputs two numbers, and the server determines the greatest using conditional logic. Demonstrates RMI with comparison operations and decision-making on the server side.

**GreatestService.java (Remote Interface):
import java.rmi.Remote;
import java.rmi.RemoteException;
 
public interface GreatestService extends Remote {
    int findGreatest(int a, int b) throws RemoteException;
}

**GreatestServiceImpl.java (Remote Object Implementation):**
```java
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
 
public class GreatestServiceImpl extends UnicastRemoteObject implements GreatestService {
    protected GreatestServiceImpl() throws RemoteException {
        super();
    }
 
    public int findGreatest(int a, int b) throws RemoteException {
        return a > b ? a : b;
    }
}

```
**GreatestServer.java (RMI Server):**
```java
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
 
public class GreatestServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099);
            GreatestServiceImpl obj = new GreatestServiceImpl();
            Naming.rebind("rmi://localhost/GreatestService", obj);
            System.out.println("Greatest Number Server Ready...");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

 


GreatestClientGUI.java (RMI Client):
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.rmi.Naming;
 
public class GreatestClientGUI extends JFrame {
    private JTextField num1Field;
    private JTextField num2Field;
    private JLabel resultLabel;
    private GreatestService service;
 
    public GreatestClientGUI() {
        setTitle("RMI Greatest of Two Numbers");
        setSize(350, 250);
        setLayout(new GridLayout(5, 2, 10, 10));
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
 
        num1Field = new JTextField();
        num2Field = new JTextField();
        JButton findButton = new JButton("Find Greatest");
        resultLabel = new JLabel("Result: ");
 
        add(new JLabel("Enter first number:"));
        add(num1Field);
        add(new JLabel("Enter second number:"));
        add(num2Field);
        add(new JLabel());
        add(findButton);
        add(new JLabel());
        add(resultLabel);
 
        try {
            service = (GreatestService) Naming.lookup("rmi://localhost/GreatestService");
        } catch (Exception e) {
            JOptionPane.showMessageDialog(this, "Error connecting to server: " + e.getMessage());
        }
 
        findButton.addActionListener(e -> {
            try {
                int a = Integer.parseInt(num1Field.getText());
                int b = Integer.parseInt(num2Field.getText());
                int result = service.findGreatest(a, b);
                resultLabel.setText("Result: " + result + " is greatest");
            } catch (Exception ex) {
                resultLabel.setText("Error: " + ex.getMessage());
            }
        });
 
        setVisible(true);
    }
 
    public static void main(String[] args) {
        new GreatestClientGUI();
    }
}
Output:
 

 


 



```
### 4.5 Number to Words Converter GUI using RMI

**Description:** This practical implements a number-to-words converter using RMI with GUI. The server converts numeric input into English words (e.g., 123 → "One Hundred and Twenty Three"). Demonstrates RMI with string manipulation, complex algorithms, and natural language processing basics.

**NumberToWordsService.java (Remote Interface):
import java.rmi.Remote;
import java.rmi.RemoteException;
public interface NumberToWordsService extends Remote {
    String convertNumberToWords(int number) throws RemoteException;
}

**NumberToWordsServiceImpl.java (Remote Object Implementation):**
```java
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
public class NumberToWordsServiceImpl extends UnicastRemoteObject implements NumberToWordsService {
    private static final String[] units = {
        "", "One", "Two", "Three", "Four", "Five", "Six", "Seven", "Eight", "Nine", "Ten",
        "Eleven", "Twelve", "Thirteen", "Fourteen", "Fifteen", "Sixteen",
        "Seventeen", "Eighteen", "Nineteen"
    };
    private static final String[] tens = {
        "", "", "Twenty", "Thirty", "Forty", "Fifty", "Sixty", "Seventy", "Eighty", "Ninety"
    };
    protected NumberToWordsServiceImpl() throws RemoteException {
        super();
    }
    private String convert(int n) {
        if (n < 20) return units[n];
        if (n < 100) return tens[n / 10] + ((n % 10 != 0) ? " " + units[n % 10] : "");
        if (n < 1000) return units[n / 100] + " Hundred" + ((n % 100 != 0) ? " and " + convert(n % 100) : "");
        if (n < 1000000) return convert(n / 1000) + " Thousand" + ((n % 1000 != 0) ? " " + convert(n % 1000) : "");
        return "Number too large";
    }
    public String convertNumberToWords(int number) throws RemoteException {
        if (number == 0) return "Zero";
        if (number < 0) return "Minus " + convert(-number);
        return convert(number);
    }
}

```
**NumberToWordsServer.java (RMI Server):**
```java
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
public class NumberToWordsServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099);
            NumberToWordsServiceImpl obj = new NumberToWordsServiceImpl();
            Naming.rebind("rmi://localhost/NumberToWordsService", obj);
            System.out.println("Number to Words Server Ready...");
        } catch (Exception e) {            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

```
**NumberToWordsClientGUI.java (RMI Client with GUI):**
```java
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.rmi.Naming;
 
public class NumberToWordsClientGUI extends JFrame {
    private JTextField numberField;
    private JLabel resultLabel;
    private NumberToWordsService service;
 
    public NumberToWordsClientGUI() {
        setTitle("RMI Number to Words Converter");
        setSize(400, 250);
        setLayout(new GridLayout(4, 2, 10, 10));
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
 
        numberField = new JTextField();
        JButton convertButton = new JButton("Convert to Words");
        resultLabel = new JLabel("Result: ");
 
        add(new JLabel("Enter a Number:"));
        add(numberField);
        add(new JLabel());
        add(convertButton);
        add(new JLabel());
        add(resultLabel);
 
        try {
            service = (NumberToWordsService) Naming.lookup("rmi://localhost/NumberToWordsService");
        } catch (Exception e) {
            JOptionPane.showMessageDialog(this, "Error connecting to server: " + e.getMessage());
        }
 
        convertButton.addActionListener(e -> {
            try {
                int num = Integer.parseInt(numberField.getText());
                String result = service.convertNumberToWords(num);
                resultLabel.setText("Result: " + result);
            } catch (Exception ex) {
                resultLabel.setText("Error: " + ex.getMessage());
            }
        });
 
        setVisible(true);
    }
 
    public static void main(String[] args) {
        new NumberToWordsClientGUI();
    }
}
 
 
 
Output:
 

 


 


```
## 5. JDBC with Remote Object Communication & RMI - Library Database

This section demonstrates integration of database operations with RMI. Shows how to create distributed database applications where the server handles database connectivity and clients access data remotely.

**Description:** This practical implements a distributed library management system using RMI and JDBC. The server connects to a MySQL database containing book information and provides remote access to book data. Clients can retrieve all books from the library database remotely. Demonstrates three-tier architecture: Client → RMI Server → Database.

**Database Schema:**
- Database: Library
- Table: Book (Book_id INT, Book_name VARCHAR, Book_author VARCHAR)

**Book.java (Serializable Data Class):
package ROC;
import java.io.Serializable;
public class Book implements Serializable {
    private int bookId;
    private String bookName;
    private String bookAuthor;
     public Book(int bookId, String bookName, String bookAuthor) {
        this.bookId = bookId;
        this.bookName = bookName;
        this.bookAuthor = bookAuthor;
    }
    public int getBookId() { return bookId; }
    public String getBookName() { return bookName; }
    public String getBookAuthor() { return bookAuthor; }
 
    public String toString() {
        return "Book ID: " + bookId + ", Title: " + bookName + ", Author: " + bookAuthor;
    }
}

**LibraryService.java (Remote Interface):**
```java
package ROC;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;
public interface LibraryService extends Remote {
    List<Book> getAllBooks() throws RemoteException;
}

```
**LibraryServer.java (RMI Server):**
```java
package ROC;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
public class LibraryServer {
    public static void main(String[] args) {
        try {
            LocateRegistry.createRegistry(1099);
            LibraryServiceImpl obj = new LibraryServiceImpl();
            Naming.rebind("rmi://localhost/LibraryService", obj);
            System.out.println("Library RMI Server Ready...");
        } catch (Exception e) {
            System.out.println("Server Error: " + e.getMessage());
        }
    }
}

```
**LibraryServiceImpl.java (Remote Object Implementation with JDBC):**
```java
package ROC;
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
 
public class LibraryServiceImpl extends UnicastRemoteObject implements LibraryService {
    Connection conn;
 
    protected LibraryServiceImpl() throws RemoteException {
        super();
        try {
            Class.forName("oracle.jdbc.driver.OracleDriver");
            conn = DriverManager.getConnection(
                "jdbc:oracle:thin:@localhost:1521:XE", "system", "nishant2711");
        } catch (Exception e) {
            System.out.println("Database connection error: " + e.getMessage());
        }
    }
 
    public List<Book> getAllBooks() throws RemoteException {
        List<Book> list = new ArrayList<>();
        try {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM Book");
            while (rs.next()) {
                list.add(new Book(rs.getInt("Book_id"), rs.getString("Book_name"), rs.getString("Book_author")));
            }
        } catch (Exception e) {
            System.out.println("Error fetching books: " + e.getMessage());
        }
        return list;
    }
}

```
**LibraryClient.java (RMI Client):**
```java
package ROC;
import java.rmi.Naming;
import java.util.List;
public class LibraryClient {
    public static void main(String[] args) {
        try {
            LibraryService service = (LibraryService) Naming.lookup("rmi://localhost/LibraryService");
            List<Book> books = service.getAllBooks();
            System.out.println("Book Information from Library Database:\n");
            for (Book b : books) {
                System.out.println(b);
            }
        } catch (Exception e) {
            System.out.println("Client Error: " + e.getMessage());
        }
    }
}
 
Output:

 

 


 
```
## 6. JDBC with Remote Object Communication & RMI - Student Database

**Description:** This practical implements a distributed student information system using RMI and JDBC. The server manages database connectivity to MySQL and provides remote access to student records. Demonstrates handling of multiple attributes and different data types (int, String, float) in distributed database applications.

**Database Schema:**
- Database: StudentDB
- Table: student_data (ID INT, NAME VARCHAR, BRANCH VARCHAR, PERCENTAGE FLOAT, EMAIL VARCHAR)

**Student.java (Serializable Data Class):
package ROC2;
import java.io.Serializable;
public class Student implements Serializable {
    private int id;    private String name;
    private String branch;    private float percentage;
    private String email;
    public Student(int id, String name, String branch, float percentage, String email) {
        this.id = id;        this.name = name;
        this.branch = branch;        this.percentage = percentage;        this.email = email;
    }
    public int getId() { return id; }
    public String getName() { return name; }
    public String getBranch() { return branch; }
    public float getPercentage() { return percentage; }
    public String getEmail() { return email; }
    public String toString() {        return "ID: " + id + ", Name: " + name + ", Branch: " + branch +
               ", Percentage: " + percentage + ", Email: " + email;
    }
}
StudentService.java (Remote Interface):
package ROC2;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;
public interface StudentService extends Remote {
    List<Student> getAllStudents() throws RemoteException;
}

**StudentServer.java (RMI Server):**
```java
package ROC2;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
public class StudentServer {
    public static void main(String[] args) {
        try {
            System.setProperty("java.rmi.server.hostname","127.0.0.1");
            LocateRegistry.createRegistry(1099);
            StudentServiceImpl obj = new StudentServiceImpl();
            Naming.rebind("rmi://127.0.0.1/StudentService", obj);
            System.out.println("Student RMI Server Ready...");
        } catch (Exception e) {            e.printStackTrace();
        }
    }
}

```
**StudentServiceImpl.java (Remote Object Implementation with JDBC):**
```java
package ROC2;
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
public class StudentServiceImpl extends UnicastRemoteObject implements StudentService {
    Connection conn;
    protected StudentServiceImpl() throws RemoteException {
        super();
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
            conn = DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/StudentDB", "root", "your_mysql_password");
            System.out.println("Database connected successfully.");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public List<Student> getAllStudents() throws RemoteException {
        List<Student> list = new ArrayList<>();
        try {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM student_data");
            while (rs.next()) {
                list.add(new Student(
                    rs.getInt("ID"),
                    rs.getString("NAME"),
                    rs.getString("BRANCH"),
                    rs.getFloat("PERCENTAGE"),
                    rs.getString("EMAIL")
                ));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return list;
    }
}

```
**StudentClient.java (RMI Client):**
```java
package ROC2;
import java.rmi.Naming;
import java.util.List;
public class StudentClient {
    public static void main(String[] args) {
        try {
            StudentService service = (StudentService) Naming.lookup("rmi://127.0.0.1/StudentService");
            List<Student> students = service.getAllStudents();
            System.out.println("Student Information from Database:\n");
            for (Student s : students) {                System.out.println(s);            }
        } catch (Exception e) {            e.printStackTrace();        }
    }
}
Output:

 

 


 


```
## 7. JDBC with Remote Object Communication & RMI - Electric Bill Database

**Description:** This practical implements a distributed electric bill management system using RMI and JDBC. The server provides remote access to billing information from MySQL database. Demonstrates handling of Date objects and financial data (float) in distributed systems.

**Database Schema:**
- Database: ElectricBillDB
- Table: Bill (consumer_name VARCHAR, bill_due_date DATE, bill_amount FLOAT)

**Bill.java (Serializable Data Class):
import java.io.Serializable;
import java.util.Date;
public class Bill implements Serializable {
    private String consumerName;
    private Date billDueDate;
    private float billAmount;
    public Bill(String consumerName, Date billDueDate, float billAmount) {
        this.consumerName = consumerName;
        this.billDueDate = billDueDate;
        this.billAmount = billAmount;
    }
    public String getConsumerName() { return consumerName; }
    public Date getBillDueDate() { return billDueDate; }
    public float getBillAmount() { return billAmount; }
    public String toString() {
        return "Consumer: " + consumerName + ", Due Date: " + billDueDate + ", Amount: " + billAmount;
    }
}

**BillService.java (Remote Interface):**
```java
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;
public interface BillService extends Remote {
    List<Bill> getAllBills() throws RemoteException;
}

```
**BillServer.java (RMI Server):**
```java
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
public class BillServer {
    public static void main(String[] args) {
        try {
            System.setProperty("java.rmi.server.hostname","127.0.0.1");
            LocateRegistry.createRegistry(1099);
            BillServiceImpl obj = new BillServiceImpl();
            Naming.rebind("rmi://127.0.0.1/BillService", obj);
            System.out.println("Bill RMI Server Ready...");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```
**BillServiceImpl.java (Remote Object Implementation with JDBC):**
```java
import java.rmi.server.UnicastRemoteObject;
import java.rmi.RemoteException;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
public class BillServiceImpl extends UnicastRemoteObject implements BillService {
    Connection conn;
    protected BillServiceImpl() throws RemoteException {
        super();
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
            conn = DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/ElectricBillDB", "root", " mysql_password");
            System.out.println("Database connected successfully.");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public List<Bill> getAllBills() throws RemoteException {
        List<Bill> list = new ArrayList<>();
        try {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM Bill");
            while (rs.next()) {
                list.add(new Bill(
                    rs.getString("consumer_name"),
                    rs.getDate("bill_due_date"),
                    rs.getFloat("bill_amount")
                ));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return list;
    }
}

```
**BillClient.java (RMI Client):**
```java
import java.rmi.Naming;
import java.util.List;
public class BillClient {
    public static void main(String[] args) {
        try {
            BillService service = (BillService) Naming.lookup("rmi://127.0.0.1/BillService");
            List<Bill> bills = service.getAllBills();
            System.out.println("Electric Bill Information:\n");
            for (Bill b : bills) {
                System.out.println(b);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```
**Output:**

---

## 8. Implementation of Mutual Exclusion using Token Ring Algorithm

**Description:** This practical implements the Token Ring algorithm for mutual exclusion in distributed systems. Multiple processes compete for access to a critical section, and only the process holding the token can enter. The token circulates among processes in a logical ring topology. Demonstrates synchronization, mutual exclusion, and fair resource allocation in concurrent systems.

**Concepts Demonstrated:**
- Mutual Exclusion in distributed systems
- Token-based synchronization
- Ring topology for process communication
- Critical section management
- Fair scheduling (each process gets equal opportunity)

**TokenRing.java (Token Manager):**
```java
class TokenRing {
    private int numProcesses;
    private volatile int tokenHolder = 0;

    public TokenRing(int numProcesses) {
        this.numProcesses = numProcesses;
    }
    public synchronized void requestToken(int processId) {
        while (processId != tokenHolder) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public synchronized void releaseToken() {
        tokenHolder = (tokenHolder + 1) % numProcesses;
        notifyAll();
    }
}

```
**Process.java (Process Thread):**
```java
class Process extends Thread {
    private int id;
    private TokenRing ring;

    public Process(int id, TokenRing ring) {
        this.id = id;
        this.ring = ring;
    }
    @Override
    public void run() {
        while (true) {
            ring.requestToken(id);
            System.out.println("Process " + id + " has entered the critical section.");
            try {
                Thread.sleep(1000); // Simulate work
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("Process " + id + " is leaving the critical section.");
            ring.releaseToken();
            try {                Thread.sleep(500);             } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```
**TokenRingMain.java (Main Application):**
```java
package prac9;

public class TokenRingMain {
    public static void main(String[] args) {
        int numProcesses = 5;
        TokenRing ring = new TokenRing(numProcesses);

        for (int i = 0; i < numProcesses; i++) {
            new Process(i, ring).start();
        }
    }
}

```
**Output:**

---

## 9. Implementation of Bully Election Algorithm

**Description:** This practical implements the Bully Election Algorithm for coordinator election in distributed systems. When a coordinator fails or a process doesn't know the current coordinator, an election is triggered. The process with the highest ID becomes the new coordinator. Demonstrates fault tolerance, leader election, and dynamic system reconfiguration.

**Concepts Demonstrated:**
- Leader election in distributed systems
- Fault detection and recovery
- Bully algorithm (highest ID wins)
- Process failure and recovery simulation
- Coordinator announcement and acknowledgment

**Process.java (Process with Election Logic):**
```java
package prac10;

public class Process extends Thread {
    int id;
    boolean isActive = true;
    static int coordinatorId = -1;
    Process[] processes;

    public Process(int id, Process[] processes) {
        this.id = id;
        this.processes = processes;
    }

    public void fail() {
        isActive = false;
        System.out.println("Process " + id + " has failed.");
        if (id == coordinatorId) {
            System.out.println("Coordinator " + id + " has failed. Election will be triggered.");
            coordinatorId = -1;
            triggerElection();
        }
    }

    public void recover() {
        isActive = true;
        System.out.println("Process " + id + " has recovered.");
        triggerElection();
    }

    public void triggerElection() {
        if (!isActive) return;
        System.out.println("Process " + id + " is starting an election...");
        boolean higherExists = false;

        for (int i = id + 1; i < processes.length; i++) {
            if (processes[i].isActive) {
                System.out.println("Process " + id + " sends election to Process " + processes[i].id);
                higherExists = true;
            }
        }

        if (!higherExists) {
            coordinatorId = id;
            System.out.println("Process " + id + " becomes the coordinator.");
            announceCoordinator();
        }
    }

    public void announceCoordinator() {
        for (Process p : processes) {
            if (p.isActive) {
                System.out.println("Process " + coordinatorId + " announces itself as coordinator to Process " + p.id);
            }
        }
    }

    @Override
    public void run() {
        try {
            while (true) {
                Thread.sleep((int)(Math.random() * 8000 + 2000)); // Random delay
                double action = Math.random();
                if (action < 0.3 && isActive) {
                    fail();
                } else if (action < 0.6 && !isActive) {
                    recover();
                } else if (action < 0.9 && isActive && coordinatorId == -1) {
                    triggerElection();
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```
**ElectionMain.java (Main Application):**
```java
package prac10;

public class ElectionMain {
    public static void main(String[] args) {
        int numProcesses = 5;

        Process[] processes = new Process[numProcesses];
        for (int i = 0; i < numProcesses; i++) {
            processes[i] = new Process(i, processes);
        }

        System.out.println("Starting automatic Bully Election simulation...\n");

        for (Process p : processes) {
            p.start();
        }
    }
}

```
**Output:**

---

## Summary

This repository covers comprehensive practical implementations in distributed systems:

1. **Socket Programming**: TCP and UDP communication for client-server applications
2. **Remote Procedure Call (RPC)**: Procedure-based remote execution
3. **Remote Method Invocation (RMI)**: Object-oriented distributed computing
4. **GUI Integration**: User-friendly interfaces with RMI backend
5. **Database Integration**: Three-tier architecture with JDBC and RMI
6. **Distributed Algorithms**: Mutual exclusion and leader election

**Technologies Used:**
- Java Socket Programming (TCP/UDP)
- Java RMI Framework
- Java Swing (GUI)
- JDBC (MySQL/Oracle Database Connectivity)
- Multi-threading and Concurrency
- Distributed Algorithms (Token Ring, Bully Election)

**Author:** Student with Roll No. C24120

---
