<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Open Search Logs UI</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f4f4f4;
        }
        .container {
            max-width: 1200px;
            margin: 20px auto;
            padding: 20px;
            background-color: #fff;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }
        .search-bar {
            display: flex;
            justify-content: space-between;
            margin-bottom: 20px;
        }
        .search-bar input {
            padding: 10px;
            width: 30%;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        .search-bar label {
            display: flex;
            align-items: center;
        }
        .search-bar button {
            padding: 10px 20px;
            background-color: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .search-bar button:hover {
            background-color: #0056b3;
        }
        .logs {
            margin-top: 20px;
            max-height: 300px;
            overflow-y: auto;
            border-top: 1px solid #ddd;
            padding-top: 10px;
        }
        .log-entry {
            background-color: #f9f9f9;
            padding: 10px;
            margin: 5px 0;
            border-radius: 4px;
            border: 1px solid #ddd;
        }
        .log-entry p {
            margin: 5px 0;
        }
        .log-level {
            font-weight: bold;
        }
        .log-level.INFO {
            color: #28a745; /* Green */
        }
        .log-level.ERROR {
            color: #dc3545; /* Red */
        }
        .log-level.WARNING {
            color: #ffc107; /* Yellow */
        }
        .loading {
            text-align: center;
            padding: 10px;
            color: #007bff;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="search-bar">
            <div>
                <label for="timestamp-search">Timestamp:</label>
                <input type="text" id="timestamp-search" placeholder="Search by Timestamp">
            </div>
            <div>
                <label for="level-search">Log Level:</label>
                <input type="text" id="level-search" placeholder="Search by Log Level">
            </div>
            <div>
                <label for="message-search">Message:</label>
                <input type="text" id="message-search" placeholder="Search by Message">
            </div>
            <button onclick="searchLogs()">Search</button>
        </div>

        <div class="logs" id="logs">
            <div id="loading" class="loading">Loading logs...</div>
            <!-- Logs will be displayed here -->
        </div>
    </div>

    <script>
        const apiUrl = "https://example.com/api/logs"; // Replace with your API URL

        let allLogs = [];

        // Function to fetch logs from the API
        async function fetchLogs() {
            try {
                const response = await fetch(apiUrl);
                const data = await response.json();
                allLogs = data.logs; // Assuming the API returns a 'logs' array
                displayLogs(allLogs);
            } catch (error) {
                console.error('Error fetching logs:', error);
                document.getElementById('loading').innerText = "Failed to load logs.";
            }
        }

        // Function to display logs in the UI
        function displayLogs(logs) {
            const logsContainer = document.getElementById('logs');
            const loadingElement = document.getElementById('loading');
            loadingElement.style.display = 'none';  // Hide loading message when logs are loaded
            logsContainer.innerHTML = '';

            if (logs.length === 0) {
                logsContainer.innerHTML = '<p>No logs found.</p>';
                return;
            }

            logs.forEach(log => {
                const logElement = document.createElement('div');
                logElement.classList.add('log-entry');
                logElement.innerHTML = `
                    <p><strong class="log-level ${log.logLevel}">${log.logLevel}</strong> - <span>${log.timestamp}</span></p>
                    <p>${log.message}</p>
                `;
                logsContainer.appendChild(logElement);
            });
        }

        // Function to search logs based on multiple criteria (timestamp, logLevel, and message)
        function searchLogs() {
            const timestampQuery = document.getElementById('timestamp-search').value.toLowerCase();
            const levelQuery = document.getElementById('level-search').value.toLowerCase();
            const messageQuery = document.getElementById('message-search').value.toLowerCase();

            const filteredLogs = allLogs.filter(log => {
                const matchesTimestamp = log.timestamp.toLowerCase().includes(timestampQuery);
                const matchesLevel = log.logLevel.toLowerCase().includes(levelQuery);
                const matchesMessage = log.message.toLowerCase().includes(messageQuery);

                return matchesTimestamp && matchesLevel && matchesMessage;
            });

            displayLogs(filteredLogs);
        }

        // Fetch logs when the page loads
        window.onload = fetchLogs;
    </script>
</body>
</html>
