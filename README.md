# Data Masking
Scripts to help mask data for automated and manual testing

Note: Faker is an excellent library to help generate dummy data to mask sensitive columns

# Step 1
Identify sensitive columns in your database that should be masked, like names, email addresses, phone numbers, social security numbers, addresses, etc.

# Step 2
Create a copy of your database that would be used for testing, you can create scripts/pipelines to generate this database on regular intervals or on-demand.

### For Postgres
- bash script (Linux/Mac OS)
```bash
#!/bin/bash
DB_NAME="your_db_name"
COPY_DB_NAME="${DB_NAME}_test_copy"

# Create a copy of the database
pg_dump $DB_NAME | psql $COPY_DB_NAME

```
- Power Shell (Windows)
```powershell
$DBName = "your_db_name"
$CopyDBName = "${DBName}_test_copy"
pg_dump $DBName | psql $CopyDBName
```

### For MySQL
- bash script (Linux/Mac OS)
```bash
#!/bin/bash
DB_NAME="your_db_name"
COPY_DB_NAME="${DB_NAME}_test_copy"
USER="your_user"
PASSWORD="your_password"

# Create a copy of the database
mysqldump -u $USER -p$PASSWORD $DB_NAME | mysql -u $USER -p$PASSWORD $COPY_DB_NAME
```
- Power Shell (Windows)
```powershell
$DBName = "your_db_name"
$CopyDBName = "${DBName}_test_copy"
$User = "your_user"
$Password = "your_password"
cmd.exe /c "mysqldump -u $User -p$Password $DBName | mysql -u $User -p$Password $CopyDBName"
```

### For SQL Server
- Power Shell (Windows)
```powershell
# PowerShell script to create a copy of a database in SQL Server
$sourceDB = "your_db_name"
$targetDB = "${sourceDB}_test_copy"
$serverInstance = "your_server_instance"
Invoke-Sqlcmd -Query "RESTORE DATABASE [$targetDB] FROM DISK='C:\Backup\$sourceDB.bak' WITH MOVE '$sourceDB' TO 'C:\Data\$targetDB.mdf', MOVE '${sourceDB}_log' TO 'C:\Data\$targetDB_log.ldf';" -ServerInstance $serverInstance
```
### For Oracle
- bash script (Linux/Mac OS)
```bash
#!/bin/bash
ORACLE_SID=your_sid
ORACLE_USER=your_user
ORACLE_PASS=your_password
# Use Data Pump Export/Import
expdp $ORACLE_USER/$ORACLE_PASS@ORACLE_SID directory=DATA_PUMP_DIR dumpfile=your_db_name.dmp logfile=expdp.log schemas=your_schema
impdp $ORACLE_USER/$ORACLE_PASS@ORACLE_SID directory=DATA_PUMP_DIR dumpfile=your_db_name.dmp logfile=impdp.log remap_schema=your_schema:your_schema_test
```

# Step 3
Once the test copy of the database has been created you can run scripts to mask sensitive columns

## NodeJS
Create a new nodejs project and add the following dependencies
```bash
npm install pg faker
```

Run the following script
```javascript
const { Client } = require('pg');
const faker = require('faker');

// Database connection configuration
const client = new Client({
  user: 'your_db_user',
  host: 'your_db_host',
  database: 'your_test_db',
  password: 'your_db_password',
  port: 5432, // Default PostgreSQL port
});

// Array of table configurations for masking
const tablesToMask = [
  {
    tableName: 'users',
    columns: {
      name: () => faker.name.findName(),
      email: () => faker.internet.email(),
    },
  },
  {
    tableName: 'contacts',
    columns: {
      phone: () => faker.phone.phoneNumber(),
      address: () => faker.address.streetAddress(),
    },
  },
];

async function maskData() {
  try {
    await client.connect();
    console.log('Connected to the database');

    for (const table of tablesToMask) {
      const { tableName, columns } = table;

      // Retrieve all rows from the table
      const res = await client.query(`SELECT id FROM ${tableName}`);
      const rows = res.rows;

      for (const row of rows) {
        // Prepare SET clause for updating columns
        const setClause = Object.keys(columns)
          .map((col) => `${col} = '${columns[col]()}'`)
          .join(', ');

        // Execute the update query
        await client.query(`UPDATE ${tableName} SET ${setClause} WHERE id = $1`, [row.id]);
        console.log(`Updated row with ID ${row.id} in table ${tableName}`);
      }
    }
  } catch (err) {
    console.error('Error occurred:', err.stack);
  } finally {
    await client.end();
    console.log('Disconnected from the database');
  }
}

maskData();
```

## Python
Create a new Python project and the following libraries
```bash
pip install psycopg2 faker
```
Run the following script
```python
import psycopg2
from faker import Faker

# Initialize Faker
faker = Faker()

# Database connection configuration
connection = psycopg2.connect(
    host="your_db_host",
    database="your_test_db",
    user="your_db_user",
    password="your_db_password",
    port=5432  # Default PostgreSQL port
)

# Array of table configurations for masking
tables_to_mask = [
    {
        "table_name": "users",
        "columns": {
            "name": lambda: faker.name(),
            "email": lambda: faker.email(),
        }
    },
    {
        "table_name": "contacts",
        "columns": {
            "phone": lambda: faker.phone_number(),
            "address": lambda: faker.street_address(),
        }
    }
]

try:
    # Open a cursor to perform database operations
    cursor = connection.cursor()
    print("Connected to the database")

    for table in tables_to_mask:
        table_name = table["table_name"]
        columns = table["columns"]

        # Retrieve all rows from the table
        cursor.execute(f"SELECT id FROM {table_name}")
        rows = cursor.fetchall()

        for row in rows:
            row_id = row[0]
            set_clause = ", ".join(
                f"{col} = %s" for col in columns.keys()
            )
            values = [func() for func in columns.values()]
            values.append(row_id)

            # Execute the update query
            cursor.execute(
                f"UPDATE {table_name} SET {set_clause} WHERE id = %s",
                values
            )
            print(f"Updated row with ID {row_id} in table {table_name}")

    # Commit changes to the database
    connection.commit()
    print("All data masked successfully")

except (Exception, psycopg2.DatabaseError) as error:
    print(f"Error occurred: {error}")
finally:
    if connection:
        cursor.close()
        connection.close()
        print("Disconnected from the database")
```

## C#
Create a .NET Console Project and add the following dependencies
```cmd
dotnet add package Npgsql
dotnet add package Bogus
```
Note: Npgsql is used for this example, but you can use the database you are targeting

Add the following code to Program.cs
```csharp
using System;
using System.Collections.Generic;
using Npgsql;
using Bogus;

class Program
{
    static void Main(string[] args)
    {
        // Database connection configuration
        string connString = "Host=your_db_host;Username=your_db_user;Password=your_db_password;Database=your_test_db";

        // Initialize Faker
        var faker = new Faker();

        // Define tables and columns to mask
        var tablesToMask = new List<TableMask>
        {
            new TableMask
            {
                TableName = "users",
                Columns = new Dictionary<string, Func<string>>
                {
                    { "name", () => faker.Name.FullName() },
                    { "email", () => faker.Internet.Email() }
                }
            },
            new TableMask
            {
                TableName = "contacts",
                Columns = new Dictionary<string, Func<string>>
                {
                    { "phone", () => faker.Phone.PhoneNumber() },
                    { "address", () => faker.Address.StreetAddress() }
                }
            }
        };

        try
        {
            using (var conn = new NpgsqlConnection(connString))
            {
                conn.Open();
                Console.WriteLine("Connected to the database");

                using (var cmd = new NpgsqlCommand())
                {
                    cmd.Connection = conn;

                    foreach (var table in tablesToMask)
                    {
                        // Retrieve all rows from the table
                        cmd.CommandText = $"SELECT id FROM {table.TableName}";
                        var reader = cmd.ExecuteReader();
                        var ids = new List<int>();

                        while (reader.Read())
                        {
                            ids.Add(reader.GetInt32(0));
                        }
                        reader.Close();

                        foreach (var id in ids)
                        {
                            var setClauses = new List<string>();
                            var parameters = new List<object>();

                            foreach (var column in table.Columns)
                            {
                                string paramName = $"@param_{column.Key}";
                                setClauses.Add($"{column.Key} = {paramName}");
                                parameters.Add(column.Value());
                                cmd.Parameters.AddWithValue(paramName, parameters[^1]);
                            }
                            cmd.Parameters.AddWithValue("@id", id);

                            // Execute the update query
                            cmd.CommandText = $"UPDATE {table.TableName} SET {string.Join(", ", setClauses)} WHERE id = @id";
                            cmd.ExecuteNonQuery();
                            cmd.Parameters.Clear();

                            Console.WriteLine($"Updated row with ID {id} in table {table.TableName}");
                        }
                    }
                }
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error occurred: {ex.Message}");
        }
        finally
        {
            Console.WriteLine("Disconnected from the database");
        }
    }

    public class TableMask
    {
        public string TableName { get; set; }
        public Dictionary<string, Func<string>> Columns { get; set; }
    }
}
```

## PHP
Create a PHP project and add the following composer dependencies
```bash
composer require fakerphp/faker
```
Run the following script
```php
<?php
require 'vendor/autoload.php';

// Database connection configuration
$host = 'your_db_host';
$dbname = 'your_test_db';
$user = 'your_db_user';
$password = 'your_db_password';
$port = 5432; // Default PostgreSQL port

try {
    // Connect to the PostgreSQL database
    $dsn = "pgsql:host=$host;port=$port;dbname=$dbname";
    $pdo = new PDO($dsn, $user, $password, [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]);
    echo "Connected to the database\n";

    // Initialize Faker
    $faker = Faker\Factory::create();

    // Array of tables and columns to mask
    $tablesToMask = [
        [
            'table_name' => 'users',
            'columns' => [
                'name' => function() use ($faker) { return $faker->name(); },
                'email' => function() use ($faker) { return $faker->email(); },
            ],
        ],
        [
            'table_name' => 'contacts',
            'columns' => [
                'phone' => function() use ($faker) { return $faker->phoneNumber(); },
                'address' => function() use ($faker) { return $faker->streetAddress(); },
            ],
        ],
    ];

    foreach ($tablesToMask as $table) {
        $tableName = $table['table_name'];
        $columns = $table['columns'];

        // Fetch all IDs from the table
        $stmt = $pdo->query("SELECT id FROM $tableName");
        $ids = $stmt->fetchAll(PDO::FETCH_COLUMN);

        foreach ($ids as $id) {
            $setClauses = [];
            $params = [];
            foreach ($columns as $column => $func) {
                $setClauses[] = "$column = ?";
                $params[] = $func();
            }
            $params[] = $id;

            // Prepare the update query
            $setClauseString = implode(', ', $setClauses);
            $updateStmt = $pdo->prepare("UPDATE $tableName SET $setClauseString WHERE id = ?");
            $updateStmt->execute($params);

            echo "Updated row with ID $id in table $tableName\n";
        }
    }

    echo "All data masked successfully\n";

} catch (PDOException $e) {
    echo "Error: " . $e->getMessage() . "\n";
} finally {
    // Close the connection
    $pdo = null;
    echo "Disconnected from the database\n";
}
```