# Index Management Guide

This document provides instructions for rebuilding indices within the SeaTicket Indexer system.

## Rebuilding Indices

Rebuilding an index is typically a two-step process: clearing the existing index and then updating it with fresh data.

### Rebuild Connection Index

Use these commands to rebuild the index for a specific connection.

1.  **Navigate to the script directory:**
    ```shell
    cd /opt/seaticket/seaticket-indexer/seaqa_indexer/script
    ```

2.  **Clear the existing index:**
    ```shell
    ./rebuild_index.sh --connection-id <CONNECTION_ID> clear
    ```

3.  **Update the index:**
    ```shell
    ./rebuild_index.sh --connection-id <CONNECTION_ID> update
    ```

### Rebuild Project Index

Use these commands to rebuild the index for a specific project.

1.  **Navigate to the script directory:**
    ```shell
    cd /opt/seaticket/seaticket-indexer/seaqa_indexer/index/script
    ```

2.  **Clear the existing index:**
    ```shell
    ./rebuild_index.sh --project-uuid <PROJECT_UUID> clear
    ```

3.  **Update the index:**
    ```shell
    ./rebuild_index.sh --project-uuid <PROJECT_UUID> update
    ```

---
*Note: Replace `<CONNECTION_ID>` and `<PROJECT_UUID>` with your actual connection ID and project UUID respectively.*
