# CSharp-SQLiteBlob-Demo

This project is the sample of SQLite Blob that is implemented by System.Data.SQLite.

We cat treat binary data as BLOB in SQLite. 
In  .NET Framework, We cat treat it as either byte-arary or SQLiteBlob.

The example of byte-array is provided in [Stackoverflow](https://stackoverflow.com/questions/625029/how-do-i-store-and-retrieve-a-blob-from-sqlite), But I can't find a memory saving example of SQLiteBLob.

So I try to make it.


## The example table, it has the blob-column.


At first, given the table that has the blob column.

```sql
create table binary_storage( 
    id  integer PRIMARY KEY, -- another alias for the rowid
    bin blob
)
```

Although we can add some column to it, If we use SQLiteDataReader.GetBlob, We should remember the bellow promise.

1. Should set "CommandBehavior.KeyInfo" to select-statement
2. Must not change the column name of the column that is blob in select-statement. Or provide an alternative of it.
3. Should not use join-statement (It's not sure).

## The example in 1.0.108.0

### Insert
```cs
long InsertDataBlob108(SQLiteConnection con, Stream inStream)
{
    var length = (int)inStream.Length;

    using (var cmd = new SQLiteCommand(con))
    {
        // At fisrst, we must reserve free space.
        // So we make new row with zeroblob.
        // see: https://sqlite.org/c3ref/blob_write.html
        cmd.CommandText = @"
            insert into binary_storage (
                bin
            ) values (
                zeroblob(@bin_len)
            )
        ";
        cmd.Parameters.Add("@bin_len", DbType.Int32).Value = length;
        cmd.ExecuteNonQuery();

        cmd.Reset();
        // In SQLite, The column that is defined as "integer PRIMARY KEY" is 
        // another alias for the rowid.
        // see: https://sqlite.org/c3ref/last_insert_rowid.html
        cmd.CommandText = "select last_insert_rowid()";
        var id = (long)cmd.ExecuteScalar();

        cmd.Reset();
        // To get SQLite Blob, We retrieve the row inserted
        // Note: This statement alias rowid to blob's column name.
	//       If a blob column is specified directly, its contents will consume  memory. 
        // see: https://sqlite.org/c3ref/blob.html
        //      https://sqlite.org/c3ref/blob_open.html
        cmd.CommandText = @"
            select rowid as bin
            from   binary_storage
            where  id = @id";
        cmd.Parameters.Add("@id", DbType.Int64).Value = id;
        using (var reader = cmd.ExecuteReader(CommandBehavior.KeyInfo))
        {
            reader.Read();

            // The replacement column of blob is set.
            using (var blob = reader.GetBlob(0, false))
            {
                byte[] buffer = new byte[BUFFER_SIZE];

                int blobOffset = 0;

                int read;
                while ((read = inStream.Read(buffer, 0, buffer.Length)) > 0)
                {
                    blob.Write(buffer, read, blobOffset);
                    blobOffset += read;
                }
            }
        }

        return id;
    }
}
```


### Select

```cs
void SelectDataBlob108(SQLiteConnection con, long id, Stream outStream)
{
    using (var cmd = new SQLiteCommand(con))
    {
        // SQLiteBlob does not provide the BLOB size.
	// This statement is get the target row and measure the BLOB size.
        cmd.CommandText = @"
            select
                rowid       as bin,
                length(bin) as bin_length
            from
                binary_storage where id = @id
        ";
        cmd.Parameters.Add("@id", DbType.Int64).Value = id;
        using (var reader = cmd.ExecuteReader(CommandBehavior.KeyInfo))
        {
            if (!reader.Read())
            {
                throw new KeyNotFoundException("id=" + id);
            }

            var blobSize = reader.GetInt32(1);

            using (var blob = reader.GetBlob(0, true))
            {
                byte[] buffer = new byte[BUFFER_SIZE];

                var blobOffset = 0;
                var blobRemain = blobSize;

                while (blobRemain > 0)
                {
                    var read = Math.Min(blobRemain, buffer.Length);
                    blob.Read(buffer, read, blobOffset);

                    outStream.Write(buffer, 0, read);
                    blobRemain -= read;
                    blobOffset += read;
                }
            }
        }
    }
}
```


## The example in 1.0.109.0 and later


At version 1.0.109.0, The library provid SQLiteBlob.Crate.

It is the wapper of [Incremental I/O](https://sqlite.org/c3ref/blob_open.html).

### Insert
```cs
long InsertDataBlob109(SQLiteConnection con, Stream inStream)
{
    var length = (int)inStream.Length;

    using (var cmd = new SQLiteCommand(con))
    {
        cmd.CommandText = @"
            insert into binary_storage (
                bin
            ) values (
                zeroblob(@bin_len)
            )
        ";
        cmd.Parameters.Add("@bin_len", DbType.Int32).Value = length;
        cmd.ExecuteNonQuery();

        cmd.Reset();
        cmd.CommandText = "select last_insert_rowid()";
        var id = (long)cmd.ExecuteScalar();

        // Incremental I/O
        // see: https://sqlite.org/c3ref/blob_open.html
        using (var blob = SQLiteBlob.Create(
            con,              // connection
            con.Database,     // database name
            "binary_storage", // table name
            "bin",            // column
            id,               // rowid
            false             // readonly
            ))
        {
            byte[] buffer = new byte[BUFFER_SIZE];

            int blobOffset = 0;

            int read;
            while ((read = inStream.Read(buffer, 0, buffer.Length)) > 0)
            {
                blob.Write(buffer, read, blobOffset);
                blobOffset += read;
            }
        }

        return id;
    }
}
```