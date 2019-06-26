# iRODS Library Examples
The goal of this repository is to provide simple examples demonstrating how to use the new libraries available in iRODS v4.2.6.

### Table of Contents
- [iRODS Query Iterator](#irods-query-iterator)
- [iRODS Connection Pool](#irods-connection-pool)
- [iRODS Thread Pool](#irods-thread-pool)
- [iRODS Filesystem](#irods-filesystem)
- [iRODS IOStreams](#irods-iostreams)
- [iRODS Query Processor](#irods-query-processor)

## iRODS Query Iterator
Demonstrates how to use `irods::query` to query the catalog.
```c++
#include <irods/irods_query.hpp>

#include <string>
#include <iostream>

void print_all_resource_names(rcComm_t& _conn)
{
    // Print all resource names known to iRODS.
    for (auto&& row : irods::query<rcComm_t>{&_conn, "select RESC_NAME"}) {
        std::cout << row[0] << '\n';
    }
}
```

## iRODS Connection Pool
Demonstrates how easy it is to create a pool of connections.
```c++
#include <irods/rodsClient.h>
#include <irods/connection_pool.hpp>

void init_connection_pool()
{
    rodsEnv env;

    if (getRodsEnv(&env) < 0) {
        // Handle error.
    }

    const int connection_pool_size = 4;
    const int refresh_time_in_secs = 600;

    // Creates a connection pool that manages 4 rcComm_t connections
    // and refreshes each connection every 600 seconds.
    irods::connection_pool pool{connection_pool_size,
                                env.rodsHost,
                                env.rodsPort,
                                env.rodsUserName,
                                env.rodsZone,
                                refresh_time_in_secs};

    // Get a connection from the pool.
    // "conn" is returned to the pool when it goes out of scope.
    // The type returned from the pool is moveable, but it cannot be copied.
    auto conn = pool.get_connection();

    // The object returned from the pool is a proxy for an rcComm_t and
    // can be implicitly cast to a reference to rcComm_t.
    rcComm_t& reference = conn;

    // Here is an example of casting to a pointer.
    // Use this for C APIs.
    auto* pointer = &static_cast<rcComm_t&>(conn);
}
```

## iRODS Thread pool
Demonstrates how to use `irods::thread_pool`.
```c++
#include <irods/thread_pool.hpp>

void schedule_task_on_thread_pool()
{
    // Creates a thread pool with 4 threads.
    // iRODS thread pool will never launch more than "std::thread::hardware_concurrency()" threads.
    irods::thread_pool pool{4};

    // This is one way to schedule a task for execution
    // "irods::thread_pool::defer" schedules the task on the thread pool. If the current thread
    // belongs to the thread pool, then the task is scheduled after the current thread returns and
    // control is returned back to the thread pool. The task is never run inside of the "defer" call.
    irods::thread_pool::defer(pool, [] {
        // Do science later!
    });

    // This is a function object.
    struct scientific_task
    {
        void operator()()
        {
            // Do science!
        }
    };

    scientific_task task;

    // This is just like "defer" except the task is scheduled immediately. The task is never
    // executed inside of the "post" call.
    irods::thread_pool::post(pool, task);

    // Only available in 4.3.0.
    // This is just like "post" except, if the current thread belongs to the thread pool, then
    // the task is executed directly inside of the call to "dispatch".
    irods::thread_pool::dispatch(pool, [] {
        // Do science later!
    });

    // Wait until ALL tasks have completed.
    // If this is not called, then on destruction of the thread pool, all tasks that have not
    // been executed are cancelled. Tasks that are still executing are allowed to finish.
    pool.join();
}
```

## iRODS Filesystem
Demonstrates how to iterate over collections as well as other functionality.
Because it implements the ISO C++17 Standard Filesystem library, you may use the documentation at [cppreference](https://cppreference.com).

Here are some helpful links:
- [iRODS Filesystem Headers](https://github.com/irods/irods/tree/4.2.6/lib/filesystem/include/filesystem)
- [Most commonly used functions](https://github.com/irods/irods/tree/4.2.6/lib/filesystem/include/filesystem/filesystem.hpp)
```c++
// If you are writing server-side code, and wish to enable the server-side api,
// you must define the following macro before including the library.
//
//    #define IRODS_FILESYSTEM_ENABLE_SERVER_SIDE_API
//
#include <irods/filesystem.hpp>

void iterating_over_collections()
{
    // IMPORTANT!!!
    // ~~~~~~~~~~~~
    // Notice that this library exists under the "experimental" namespace.
    // This is important if you're considering using this library. It means that any
    // library under this namespace could change in the future. Changes are likely
    // to only occur based on feedback from the community.
    namespace fs = irods::experimental::filesystem;

    // iRODS Filesystem has two namespaces, client and server.
    // Not all classes and functions require the use of these namespaces.

    try {
        // Here's an example of how to iterate over a collection on the client-side.
        // Notice how the "client" namespace follows the "fs" namespace alias.
        // This is required by some functions and classes to control which implementation
        // should be used. If you wanted to do this on the server-side, you would replace
        // "client" with "server". This does not recurse into subcollections.
        for (auto&& e : fs::client::collection_iterator{conn, "/path/to/collection"}) {
            // Do something with the collection entry.
        }

        // To recursively iterate over a collection and all of it's children, use a
        // recursive iterator.
        for (auto&& e : fs::client::recursive_collection_iterator{conn, "/path/to/collection"}) {
            // Do something with the collection entry.
        }

        // These iterators support shallow copying. This means, if you copy an iterator,
        // subsequent operations on the copy, such as iterating to the next entry, will
        // be visible to the original iterator.

        //
        // Let's try something new.
        //
        
        // How about getting the size of a data object.
        auto size = fs::client::data_object_size(conn, "/path/to/data_object");

        // Or checking if an object exists.
        if (fs::exists(conn, path)) {
            // Do something with it.
        }
    }
    catch (const fs::filesystem_error& e) {
        // Handle error.
    }
}
```

## iRODS IOStreams
Demonstrates how to use `dstream` and `default_transport` to read and write data objects.
```c++
// Defines 3 classes:
// - idstream: Input-only stream for reading data objects.
// - odstream: Output-only stream for writing data objects.
// - dstream : Bidirectional stream for reading and writing data objects.
#include <irods/dstream.hpp>

// Defines the default transport mechanism for transporting bytes via the iRODS protocol.
#include <irods/transport/default_transport.hpp>

void write_to_data_object()
{
    // IMPORTANT!!!
    // ~~~~~~~~~~~~
    // Notice that this library exists under the "experimental" namespace.
    // This is important if you're considering using this library. It means that any
    // library under this namespace could change in the future. Changes are likely
    // to only occur based on feedback from the community.
    namespace io = irods::experimental::io;

    auto conn = // Our connection to iRODS.

    // Instantiates a new transport object which uses the iRODS protocol to read and
    // write bytes into a data object. Transport objects are designed to be used by IOStreams
    // objects such as dstream. "default_transport" is a wrapper around the iRODS C API for
    // reading and writing data objects.
    //
    // You can add support for more transport protocols by implementing the following interface:
    //
    //     https://github.com/irods/irods/tree/4.2.6/lib/core/include/transport/transport.hpp
    //
    io::client::default_transport xport{conn};

    // Here, we are creating a new output stream for writing. If the data object exists, then
    // the existing data object is opened, else a new data object is created.
    // We could have also used "dstream" itself, but then we'd need to pass in openmode flags
    // to instruct iRODS on how to open the data object.
    io::odstream out{xport, "/path/to/data_object"};

    if (!out) {
        // Handle error.
    }

    std::array<char, 4_Mb> buffer{}; // Buffer full of data.

    // This is the fastest way to write data into iRODS via the new stream API.
    // Data pushed through the read/write functions is unformatted as written as is.
    out.write(buffer.data(), buffer.size());

    // This will also write data into the data object. This is slower than the previous method
    // because stream operators format data.
    out << "Here is some more data ...\n";
}

void read_from_data_object()
{
    namespace io = irods::experimental::io;

    auto conn = // Our connection to iRODS.

    // See function above for information about this type.
    io::client::default_transport xport{conn};

    // Here, we are creating a new input stream for reading. 
    io::idstream in{xport, "/path/to/data_object"};

    if (!in) {
        // Handle error.
    }

    std::array<char, 4_Mb> buffer{}; // Buffer to hold data.

    // This is the fastest way to write data into iRODS via the new stream API.
    // Data pushed through the read/write functions is unformatted as written as is.
    in.read(buffer.data(), buffer.size());

    // Read a single character sequence into "word".
    // This assumes the input stream contains a sequence of printable characters.
    std::string word;
    in >> word;

    std::string line;
    while (std::getline(in, line)) {
        // Read every line of the input stream until eof.
    }
}
```

## iRODS Query Processor
Demonstrates how to use `irods::query_processor`.
```c++
#include <irods/query_processor.hpp>

#include <iostream>
#include <vector>

void process_all_query_results()
{
    // This will hold all data object absolute paths found by the
    // query processor.
    std::vector<std::string> paths;

    using query_processor = irods::query_processor<rcComm_t>;

    // This is where we create our query processor. As you can see, we pass it
    // the query string as well as the handler. The handler takes in a single row
    // (i.e. std::vector<std::string>) and creates a path using the results which
    // are stored in the referenced "paths" container.
    query_processor qproc{"select COLL_NAME, DATA_NAME", [&paths](const auto& _row) {
        paths.push_back(_row[0] + '/' + _row[1]);
    }};

    auto thread_pool = // Assume we have one of these.
    auto conn = // Assume we have one of these.

    // This is how we run the query. Notice how the execute call accepts a thread
    // pool and connection. This allows developers to run queries on different
    // thread pools.
    //
    // The object returned is a handle to a std::future containing error information.
    // By doing this, the execution and handling of the query and it's results are done
    // asynchronously, therefore the application is not blocked from doing other work.
    auto errors = qproc.execute(thread_pool, conn);

    // Becausing the errors are returned via a std::future, calling ".get()" will cause
    // the application to wait until all query results have been processed by the
    // handler provided on construction.
    for (auto&& error : errors.get()) {
        // Handle errors.
    }

    // Print all the results.
    for (auto&& path : paths) {
        std::cout << "path: " << path << '\n';
    }
}
```
