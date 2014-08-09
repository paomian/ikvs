## 实现一个kv数据库-第四部分 API设计

@(翻译)[kvdb|MarkDown]

- 这是IKVS系列的第四部分，你也可以通过文章列表选择其他本系列的部分。  

- 我终于为我的kv数据库项目选定了一个名字，从现在开始他叫FelixDB。  

- 在本文中，我将要看看这四个kv数据库和数据库系统的api：LevelDB，Kyoto Cabinet，BerkekeyDB and SQLite3。对于它们的api的每个主要功能，我会对比命名方式和方法原型平衡利弊，并为我正在开发的kv数据库FelixDB设计一套api，本文将介绍：

1. API设计的一般原则。
2. 定义功能性的FelixDB的公共API
3. 比较现有数据库的API  
    3.1 打开和关闭数据库  
    3.2 读取和写入  
    3.3 迭代  
    3.4 参数化  
    3.5 错误管理  
4. 结论
5. 参考文献

###1. API设计的一般原则

- 设计一个好的API是困难的，真的困难，但我不是要说什么的新的东西，只是复述我以前已经讲过的很多东西。迄今为止，我发现最佳的关于API设计的资源是Jonshua Bloch的格言式的报告“如何设计好的API和他为什么那么重要”。如果你还没有机会看这次报告，我强烈建议你花时间把他看完，在报告过程中，Bloch明确指出听众应该记得的两个重要的事情，我抄了报告的这些点，并增加了一些观点。  

    1. 如果有疑问，抛弃他，如果不确定API中是否要包含一个功能，类，方法，参数，就不要包含他。
    2. 不要让客户端做任何库可以做的事，如果你的API让客户端执行了一系列上一个函数调用和插件的结果是下一个的输入，只需要在你的API中添加一个函数来做一系列的函数调用。  


- 针对良好的API设计的资源有在Scott Meyers写的`Effective C++`和Joshua Bloch写的`Effective Java`的第四章设计与声明。  

- 这些资源对现阶段的kv数据库项目是非常重要的，但我认为要考虑到不包含这些资源的一个重要的因素是很重要的：用户期望。从头设计一个api太鸡吧难了，kv数据库就不一样，有历史，设计api容易多了。用户一直在使用其他kv数据库和数据库系统的API，因此，当用户面对一个新的kv数据库的API是，它们期望这些是以前所熟悉的，而不是要增加学习曲线的去遵循不熟悉的API，而且这会使它们恼怒。  

- 出于这个原因，即使我知道上述我列出来所有资源中的好的意见，我也会考虑我将尽可能的复制现有库的API，我创建这样的API的优势是因为这将使用户更容易的使用它们。
    
###2. 为FelixDB公共API定义功能

- 既然这是实现一个最小的切稳定的kv数据库的第一步，我肯定不会提供成熟项目如Kyoto Cabinet和LevelDB中所有先进的功能。我想先加入基本功能，然后逐渐增加其他功能。对我来说，基本功能就限制在这些：

    - 打开关闭数据库
    - 数据库中读写
    - 遍历一个数据库中的所有的kv集合
    - 提供一种方法来调整参数
    - 提供一个易懂的错误通知接口
    
- 我知道这些功能的用例很有限，但是对现在来说这足够了。我决定不包含任何的事物处理机制，分组查询，和原子操作。此外，我现在还不会提供快找功能。

###3. 比较现有数据库的API

- 为了比较现有数据库的C++API，我将抽取每个功能的代码来做比较。这些样本代码有的是自己所想，有的是直接取自官方文档：
    
    - Kyoto Cabinet的基本说明
    - LevelDB的详细文档
    - Berkeley DB入门
    - 在五分钟之内学会SQLite
    
- 我也使用了代码配色，以便于区分各种API代码。

    ####3.1 打开关闭数据库
    
    1. 下面是代码示例演示如何打开正在探究的系统数据库。为了代码简洁，设置和错误管理此处没有展示，并在下面相应的章节中更详细进行说明。
    
    ```cpp
    /* LevelDB */
    leveldb::DB* db;
    leveldb::DB::Open(leveldb::Options, "/tmp/testdb", &db);
    ...
    delete db;
    ```
    
    ```cpp
    /* Kyoto Cabinet */
    HashDB db;
    db.open("dbfile.kch", HashDB::OWRITER | HashDB::OCREATE);
    ...
    db.close()
    ```
    
    ```cpp
    /* SQLite3 */
    sqlite3 *db;
    sqlite3_open("askyb.db", &db);
    ...
    sqlite3_close(db);
    ```
    
    ```cpp
    /* Berkeley DB */
    Db db(NULL, 0);
    db.open(NULL, "my_db.db", NULL, DB_BTREE, DB_CREATE, 0);
    ...
    db.close(0);
    ```
    
    2. 出现了两个清晰的数据库打开方式，一方面，LevelDB和SQLite3的API需要创建一个数据库对象指针，然后调用一个以数据库对象指针为参数的打开函数，来分配内存和设置数据库，另一方面，Kyoto Cabinet和Berkeley DB的API是首先实例化一个数据库对象，然后调用该对象的open()方法来设置设置和打开数据库。
    
    3. 现在在考虑关闭部分，LevelDB只需要对指针进行`delete`操作，而对于SQLite3，是调用一个关闭函数，Kyoto Cabinet和BerkeleyDB需要调用数据库对象本身的close()方法。
    
    4. 我想想，LevelDB和SQLite3强制使用一个数据库对象指针，然后传递给一个数据库开启函数是非常C风格的设计。此外，我认为，LevelDB用对指针的`delete`操作来处理数据库关闭是一个设计上的缺陷，因为这破坏了对称性。在API中，函数调用应该尽可能的对称，因为这更直观更合乎逻辑，如果我调用了`call()`,然后我应该调用`close()`比我调用了`open()`，然后我在`delete`指针要好的多并且更符合逻辑。
    
**设计决定**
    
- 所以，我将在FelixDB中使用他，我想使用类似Kyoto Cabinet和BerkeleyDB那样直接实例化一个数据库对象，然后调用`open()`和`close()`方法，至于命名，我会坚持传统的`open()`和`close()`。
    
    ####3.2 读和写
    
    在本节中，我比较的API中读/写功能的优劣。

    ```cpp
    /* LevelDB */
    std::string value;
    db->Get(leveldb::ReadOptions(), "key1", &value);
    db->Put(leveldb::WriteOptions(), "key2", value);
    ```
    
    ```cpp
    /* Kyoto Cabinet */
    string value;
    db.get("key1", &value);
    db.set("key2", "value");
    ```
    
    ```cpp
    /* SQLite3 */
    int szErrMsg;
    char *query = “INSERT INTO table col1, col2 VALUES (‘value1’, ‘value2’)”;
    sqlite3_exec(db, query, NULL, 0, &szErrMsg);
    ```
    
    ```cpp
    /* Berkeley DB */
    /* reading */
    Dbt key, data;

    key.set_data(&money);
    key.set_size(sizeof(float));

    data.set_data(description);
    data.set_ulen(DESCRIPTION_SIZE + 1);
    data.set_flags(DB_DBT_USERMEM);

    db.get(NULL, &key, &data, 0);

    /* writing */
    char *description = "Grocery bill.";
    float money = 122.45;

    Dbt key(&money, sizeof(float));
    Dbt data(description, strlen(description) + 1);

    db.put(NULL, &key, &data, DB_NOOVERWRITE);

    int const DESCRIPTION_SIZE = 199;
    float money = 122.45;
    char description[DESCRIPTION_SIZE + 1];
    ```

    1. 在这里我不会去考虑SQLite3，因为他是基于SQL，因此他的读和写都是通过SQL语句，而不是方法调用。Berkeley DB的需要创建DBT对象，并要设置很多选项，所以我也不会去考虑他。
    
    2. 留给我们的还有LevelDB和Kyoto Cabinet，他们有额和你好的`getter/setter`对称的接口，LevelDB有`Get()`和`Put()`，Kyoto Cabinet有`get()`和`set()`。`Put()`和`set()`方法的运行非常相似，key是值传递的，该值是通过作为指针来传递，因此他可以通过调用来更新，这里该值不会在调用时被返回，返回值只是为了错误管理。
