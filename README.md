<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Logs Table</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f4f4f9;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            margin: 20px 0;
            box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
        }

        th, td {
            padding: 12px 15px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }

        th {
            background-color: #007bff;
            color: white;
        }

        tr:nth-child(even) {
            background-color: #f2f2f2;
        }

        tr:hover {
            background-color: #f1f1f1;
        }

        .container {
            max-width: 1200px;
            margin: 0 auto;
        }

        .pagination {
            display: flex;
            justify-content: center;
            margin: 20px 0;
        }

        .pagination button {
            padding: 5px 10px;
            margin: 0 2px;
            cursor: pointer;
            border: 1px solid #ddd;
            background-color: #f9f9f9;
        }

        .pagination button:hover {
            background-color: #ddd;
        }

        .filter-container {
            margin-bottom: 20px;
        }

        .filter-container label {
            margin-right: 10px;
        }

        .filter-container select {
            padding: 5px;
        }

        .search-container {
            margin-bottom: 20px;
        }

        .search-container input {
            padding: 5px 10px;
            width: 200px;
            margin-left: 10px;
        }
    </style>
</head>
<body>

    <div class="container">
        <h1>Log Table</h1>

        <!-- Filter Options -->
        <div class="filter-container">
            <label for="log-level">Filter by Log Level:</label>
            <select id="log-level" onchange="filterLogs()">
                <option value="">All</option>
                <option value="INFO">INFO</option>
                <option value="ERROR">ERROR</option>
                <option value="WARNING">WARNING</option>
            </select>
        </div>

        <!-- Search Option -->
        <div class="search-container">
            <label for="search-input">Search Logs:</label>
            <input type="text" id="search-input" oninput="searchLogs()" placeholder="Search by timestamp, level, or message" />
        </div>

        <!-- Logs Table -->
        <table id="logs-table">
            <thead>
                <tr>
                    <th>Timestamp</th>
                    <th>Log Level</th>
                    <th>Message</th>
                </tr>
            </thead>
            <tbody>
                <!-- Log rows will be added here dynamically -->
            </tbody>
        </table>

        <!-- Pagination Controls -->
        <div class="pagination" id="pagination">
            <!-- Pagination buttons will be added here -->
        </div>
    </div>

    <script>
        // Sample API endpoint, replace with your actual API endpoint
        const apiUrl = 'https://example.com/api/logs';

        let logs = [];
        let filteredLogs = [];
        let currentPage = 1;
        const logsPerPage = 5;

        // Fetch logs from the API
        async function fetchLogs() {
            try {
                const response = await fetch(apiUrl);
                if (!response.ok) {
                    throw new Error('Failed to fetch logs');
                }

                logs = await response.json();
                filteredLogs = [...logs]; // Initially no filtering

                renderTable();
                renderPagination();
            } catch (error) {
                console.error('Error fetching logs:', error);
            }
        }

        // Render the table with the current logs
        function renderTable() {
            const tableBody = document.querySelector('#logs-table tbody');
            tableBody.innerHTML = '';

            const startIdx = (currentPage - 1) * logsPerPage;
            const endIdx = startIdx + logsPerPage;

            const logsToDisplay = filteredLogs.slice(startIdx, endIdx);

            logsToDisplay.forEach(log => {
                const row = document.createElement('tr');
                const timestampCell = document.createElement('td');
                timestampCell.textContent = log.timestamp;
                row.appendChild(timestampCell);

                const logLevelCell = document.createElement('td');
                logLevelCell.textContent = log.logLevel;
                row.appendChild(logLevelCell);

                const messageCell = document.createElement('td');
                messageCell.textContent = log.message;
                row.appendChild(messageCell);

                tableBody.appendChild(row);
            });
        }

        // Render pagination controls
        function renderPagination() {
            const paginationDiv = document.getElementById('pagination');
            paginationDiv.innerHTML = '';

            const totalPages = Math.ceil(filteredLogs.length / logsPerPage);

            for (let i = 1; i <= totalPages; i++) {
                const button = document.createElement('button');
                button.textContent = i;
                button.onclick = () => changePage(i);
                if (i === currentPage) {
                    button.disabled = true;
                }
                paginationDiv.appendChild(button);
            }
        }

        // Change the current page
        function changePage(pageNumber) {
            currentPage = pageNumber;
            renderTable();
            renderPagination();
        }

        // Filter the logs by selected log level
        function filterLogs() {
            const selectedLogLevel = document.getElementById('log-level').value;

            if (selectedLogLevel) {
                filteredLogs = logs.filter(log => log.logLevel === selectedLogLevel);
            } else {
                filteredLogs = [...logs];
            }

            currentPage = 1; // Reset to first page after filtering
            renderTable();
            renderPagination();
        }

        // Search logs by timestamp, log level, or message
        function searchLogs() {
            const searchQuery = document.getElementById('search-input').value.toLowerCase();

            filteredLogs = logs.filter(log => {
                return log.timestamp.toLowerCase().includes(searchQuery) ||
                       log.logLevel.toLowerCase().includes(searchQuery) ||
                       log.message.toLowerCase().includes(searchQuery);
            });

            currentPage = 1; // Reset to first page after searching
            renderTable();
            renderPagination();
        }

        // Call fetchLogs when the page loads
        window.onload = fetchLogs;
    </script>

</body>
</html>
