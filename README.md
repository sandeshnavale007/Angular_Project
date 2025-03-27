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

        .column-search-container {
            margin-bottom: 20px;
        }

        .column-search-container label {
            margin-right: 10px;
        }

        .column-search-container input {
            padding: 5px 10px;
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

        <!-- Column-Wise Search Options -->
        <div class="column-search-container">
            <label for="search-timestamp">Search by Timestamp:</label>
            <input type="text" id="search-timestamp" oninput="searchLogs()" placeholder="Search by timestamp" />
            
            <label for="search-level">Search by Log Level:</label>
            <input type="text" id="search-level" oninput="searchLogs()" placeholder="Search by log level" />
            
            <label for="search-message">Search by Message:</label>
            <input type="text" id="search-message" oninput="searchLogs()" placeholder="Search by message" />
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
            <!-- Next and Previous buttons will be added here -->
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

        // Render pagination controls (with Next and Previous buttons)
        function renderPagination() {
            const paginationDiv = document.getElementById('pagination');
            paginationDiv.innerHTML = '';

            const totalPages = Math.ceil(filteredLogs.length / logsPerPage);

            // Show "Previous" button only if we're not on the first page
            if (currentPage > 1) {
                const prevButton = document.createElement('button');
                prevButton.textContent = 'Previous';
                prevButton.onclick = () => changePage(currentPage - 1);
                paginationDiv.appendChild(prevButton);
            }

            // Show "Next" button only if we're not on the last page
            if (currentPage < totalPages) {
                const nextButton = document.createElement('button');
                nextButton.textContent = 'Next';
                nextButton.onclick = () => changePage(currentPage + 1);
                paginationDiv.appendChild(nextButton);
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
            const searchTimestamp = document.getElementById('search-timestamp').value.toLowerCase();
            const searchLevel = document.getElementById('search-level').value.toLowerCase();
            const searchMessage = document.getElementById('search-message').value.toLowerCase();

            filteredLogs = logs.filter(log => {
                return log.timestamp.toLowerCase().includes(searchTimestamp) &&
                       log.logLevel.toLowerCase().includes(searchLevel) &&
                       log.message.toLowerCase().includes(searchMessage);
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
