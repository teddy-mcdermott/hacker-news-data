# Hacker News Data Scraper

A high-performance, parallelised data scraper that downloads the entire history of Hacker News items (stories, comments, jobs, etc.) and stores them in a local PostgreSQL database. Uses a resilient dispatcher/worker architecture for fast, efficient, and fault-tolerant downloading.

---

### Features
- **Parallel**: Uses Python's multiprocessing to run multiple worker processes, maximising CPU and network usage.
<br>

- **Asynchronous Workers**: Each worker uses asyncio and aiohttp to handle hundreds of concurrent API requests, dramatically increasing download speed.
<br>

- **Resilient Job Queue**: A PostgreSQL-backed job queue ensures that if the script is stopped or crashes, it can resume exactly where it left off with no data loss[^Lying] or duplication.

[^Lying]: There may be ~0.01% data loss as the current jobs will not be saved, however this will be recovered immediately when next downloading, this data will be the front of the job queue.

- **Efficient Database Storage**: Uses highly optimised, batched database inserts (asyncpg) to handle a high volume of writes without overwhelming the database. 
<br>
- **Real-time Monitoring**: The dispatcher provides a live, updating progress bar showing the percentage of data chunks completed.
<br>
- **Dockerised Database**: The PostgreSQL database runs in a Docker container for easy setup, portability, and cleanup.

---

### Architecture
The system is built on a dispatcher/worker model:

docker-compose.yml: Defines and runs the PostgreSQL database service in a Docker container, ensuring a consistent and isolated environment.

dispatcher.py: The main control script. On its first run, it populates a job_chunks table in the database with the entire range of Hacker News item IDs to be downloaded. It then launches a pool of worker processes and monitors the overall progress.

worker.py: The workhorse. Each worker process connects to the database, atomically claims a "chunk" of work from the job_chunks table, and then uses asynchronous requests to download all items in that range concurrently. The results are written to the database in large, efficient batches. The entire database should be retrievable in a few hours, dependent on your network speed. (for me this was ~3 hrs on 300 mbps connection with sensible settings).

---

### Prerequisites

Before you begin, ensure you have the following installed:

- Docker: To run the PostgreSQL database container.
- Python 3.10+: For running the scripts.
- Git: For cloning the repository.

### System Requirements
**Minimum**:
4 GB RAM
50 GB free disk space
Stable internet connection

**Recommended**:
8+ GB RAM
100+ GB disk space (for full history + indexes)
Multi-core CPU (4+ cores)

---
## Setup and Installation

1.  **Clone the Repository**
    ```bash
    git clone https://github.com/wayworm/hacker-news-data
    cd hacker-news-data
    ```

2.  **Install Python Dependencies**
    It's recommended to use a virtual environment.
    ```bash
    python3 -m venv venv
    source venv/bin/activate  # On Windows: venv\Scripts\activate
    pip install requests psycopg2-binary aiohttp asyncpg
    ```

3.  **Start the Database Server**
    This command will download the PostgreSQL image and start the database container in the background.
    ```bash
    docker-compose up -d
    ```

---

## How to Run

The entire process is managed by the dispatcher script.

1.  **Start the Download**
    From the project directory, run the dispatcher. It will automatically populate the job queue on the first run and then launch the workers.
    ```bash
    python dispatcher.py
    ```

2.  **Stopping the Download**
    Press `Ctrl+C` in your terminal to stop the download. Progress is automatically saved to the database, and you can resume by running `dispatcher.py` again.

3.  **Resetting the Database (Optional)**
    If you want to start the download from scratch and delete all existing data, use the `--reset-db` flag.
    ```bash
    python dispatcher.py --reset-db
    ```

---

## Configuration

You can tune the performance by adjusting the constants at the top of the `dispatcher.py` and `worker.py` files.

-   **In `dispatcher.py`:**
    -   `NUM_WORKERS`: The number of worker processes to launch. A good starting point is 1.5x the number of your CPU cores.
    -   `CHUNK_SIZE`: The number of item IDs in each job. Larger chunks mean less job management overhead. If they're too large, you do risk needed to redownload a significant amount of data if an error occurs, the data is only saved when a working finishes the chunk.

-   **In `worker.py`:**
    -   `CONCURRENT_REQUESTS`: The number of API requests a single async worker will make simultaneously. This is the most powerful dial for performance. `300` is a good value.

    -   `BATCH_SIZE`: The number of downloaded items to collect in memory before writing them to the database in a single batch. Larger batches are more efficient.

---

## Performance

On a typical setup (4-core CPU, 100 Mbps connection):
- **Download speed:** ~500-1000 items/second
- **Time to completion:** ~3-6 hours for full history
- **Database size:** ~50+ GB (varies with indexes and data volume)

---

## Database Management

You can connect to your database to view the data using any standard SQL client, like **DBeaver** or **pgAdmin**.

-   **Connection Details:**
    -   **Host**: `localhost`
    -   **Port**: `5432`
    -   **Database**: `hacker_news`
    -   **Username**: `myuser`
    -   **Password**: `mypassword`

### Data Schema

The `items` table contains:
- `id`: Unique item identifier (bigint)
- `type`: Item type (story, comment, job, poll, pollopt)
- `by`: Username of submitter
- `time`: Unix timestamp
- `text`: Content (for comments)
- `title`: Title (for stories)
- `url`: External link (for stories)
- `score`: Points/karma
- `kids`: JSONB array of child item IDs
- `deleted`: Boolean flag
- `dead`: Boolean flag

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add new feature'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

---

## References

- [Hacker News API Documentation](https://github.com/HackerNews/API)
- [PostgreSQL](https://www.postgresql.org/)
- [DBeaver](https://dbeaver.io/)
- [Docker](https://www.docker.com/)
---

## Acknowledgments

- Thanks to the Hacker News team for providing a free, open API!

