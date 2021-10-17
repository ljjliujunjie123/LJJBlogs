## **一、基本概念**

ContentProvider是Android系统中提供的专门用户不同应用间进行数据共享的组件，提供了一套标准的接口用来获取以及操作数据，准许开发者把自己的应用数据根据需求开放给其他应用进行增删改查，而无须担心直接开放数据库权限而带来的安全问题。系统预置了许多ContentProvider用于获取用户数据，比如消息、联系人、日程表等。

## **二、ContentResolver**

在ContentProvider的使用过程中，需要借用ContentResolver来控制ContentProvider所暴露处理的接口，作为代理来间接操作ContentProvider以获取数据。
在 Context.java 的源码中如下抽象方法

```java
    /** Return a ContentResolver instance for your application's package. */
    public abstract ContentResolver getContentResolver();
```

所以可以在所有继承Context的类中通过 getContentResovler() 方法获取ContentResolver

```java
ContentResolver contentResolver = getContentResovler();
```
ContentProvider中的几个抽象方法在ContentResolver中均有着一一对应的同名方法

## **三、继承ContentProvider**

创建一个自定义ContentProvider的方式是继承ContentProvider类并实现其六个抽象方法
例如

```java
public class MyContentProvider extends ContentProvider {

    public MyContentProvider() {
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        // Implement this to handle requests to delete one or more rows.
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public String getType(Uri uri) {
        // TODO: Implement this to handle requests for the MIME type of the data
        // at the given URI.
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        // TODO: Implement this to handle requests to insert a new row.
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public boolean onCreate() {
        // TODO: Implement this to initialize your content provider on startup.
        return false;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        // TODO: Implement this to handle query requests from clients.
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection,
                      String[] selectionArgs) {
        // TODO: Implement this to handle requests to update one or more rows.
        throw new UnsupportedOperationException("Not yet implemented");
    }
    
}

```
之后，需要在AdnroidManifest.xml中对ContentProvider进行注册

```java
        <provider
            android:name=".MyContentProvider"
            android:authorities="com.czy.contentprovideremo.MyContentProvider"
            android:exported="true" />
```

 - name：ContentProvider的全称类名
 - authorities：唯一标识了一个ContentProvider，外部应用通过该属性值来访问我们的ContentProvider。因此该属性值必须是唯一的，建议在命名时以包名为前缀
 - exported：表明是否允许其他应用调用ContentProvider，true表示支持，false表示不支持。默认值根据开发者的属性设置而会有所不同，如果包含 Intent-Filter 则默认值为true，否则为false

ContentProvider的六个抽象方法的含义分别是：

 - onCreate()：代表ContentProvider的创建，可以用来进行一些初始化操作
 - getType(Uri uri)：用来返回一个Uri请求所对应的MIME类型
 - insert(Uri uri, ContentValues values)：插入数据
 - query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)：查询数据
 - update(Uri uri, ContentValues values, String selection, String[] selectionArgs)：更新数据
 - delete(Uri uri, String selection, String[] selectionArgs)：删除数据

## **四、URI**

观察MyContentProvider中的几个方法，可以发现除了 onCreate() 方法外，其它五个抽象方法都包含了一个Uri（统一资源标识符）参数，通过这个对象可以来匹配对应的请求。
URI 代表了要操作的数据对象，主要包含了两部分信息：

 - 需要操作的哪个ContentProvider
 - 对ContentProvider中的哪些数据进行操作

一个URI由以下几个部分组成：

 - Scheme：ContentProvider的Scheme被规定为“content://”
 - Authority：用于唯一标识某个ContentProvider，外部调用者根据这个标识来指定要操作的ContentProvider
 - Path：用来标识要操作的具体数据

比如，ContentProvider中操作的数据可以都是从SQLite数据库中获取的，而数据库中可能存在许多张表，这时候就需要用到Uri来表明是要操作哪个数据库、操作数据库的哪张表了
那么如何确定一个Uri是要执行哪项操作呢？这里需要用到UriMatcher来帮助ContentProvider匹配Uri，它仅包含了两个方法：

 - addURI(String authority, String path, int code)：用于向UriMatcher对象注册Uri。authtity是在AndroidManifest.xml中注册的ContentProvider的authority属性值；path表示一个路径，可以设置为通配符，#表示任意数字，*表示任意字符；两者组合成一个Uri，而code则代表该Uri对应的标识码
 - match(Uri uri)：匹配传入的Uri。返回addURI()方法中传递的code参数，如果找不到匹配的标识码则返回-1

## **五、小Demo**

这里来做一个小Demo，自定义一个ContentProvider，然后向其写入和读取数据
使用SQLite作为ContentProvider的数据存储地址和数据来源，因此需要先建立一个SQLiteOpenHelper
创建一个名为"book.db"的数据库，包含“book”和“user”两张表

```java
/**
 * 作者： leavesc
 * 时间： 2017/4/3 12:07
 * 描述：
 */
public class DbOpenHelper extends SQLiteOpenHelper {

    //数据库名
    private static final String DATA_BASE_NAME = "book.db";

    //数据库版本号
    private static final int DATE_BASE_VERSION = 1;

    //表名-书
    public static final String BOOK_TABLE_NAME = "book";

    //表名-用户
    public static final String USER_TABLE_NAME = "user";

    //创建表-书（两列：主键自增长、书名）
    private final String CREATE_BOOK_TABLE = "create table " + BOOK_TABLE_NAME
            + "(_id integer primary key autoincrement, bookName text)";

    //创建表-用户（三列：主键自增长、用户名、性别）
    private final String CREATE_USER_TABLE = "create table " + USER_TABLE_NAME
            + "(_id integer primary key autoincrement, userName text, sex text)";

    public DbOpenHelper(Context context) {
        super(context, DATA_BASE_NAME, null, DATE_BASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(CREATE_BOOK_TABLE);
        db.execSQL(CREATE_USER_TABLE);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

    }

}
```
自定义ContentProvider
当中，getTableName(Uri uri)方法用于判定Uri指向的数据库表名
然后在initProviderData()方法中向数据库插入一些原始数据

```java
public class BookProvider extends ContentProvider {

    private Context context;

    private SQLiteDatabase sqLiteDatabase;

    public static final String AUTHORITY = "com.czy.contentproviderdemo.BookProvider";

    public static final int BOOK_URI_CODE = 0;

    public static final int USER_URI_CODE = 1;

    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        uriMatcher.addURI(AUTHORITY, DbOpenHelper.BOOK_TABLE_NAME, BOOK_URI_CODE);
        uriMatcher.addURI(AUTHORITY, DbOpenHelper.USER_TABLE_NAME, USER_URI_CODE);
    }

    private String getTableName(Uri uri) {
        String tableName = null;
        switch (uriMatcher.match(uri)) {
            case BOOK_URI_CODE:
                tableName = DbOpenHelper.BOOK_TABLE_NAME;
                break;
            case USER_URI_CODE:
                tableName = DbOpenHelper.USER_TABLE_NAME;
                break;
        }
        return tableName;
    }

    public BookProvider() {

    }

    @Override
    public boolean onCreate() {
        context = getContext();
        initProviderData();
        return false;
    }

    //初始化原始数据
    private void initProviderData() {
        sqLiteDatabase = new DbOpenHelper(context).getWritableDatabase();
        sqLiteDatabase.beginTransaction();
        ContentValues contentValues = new ContentValues();
        contentValues.put("bookName", "数据结构");
        sqLiteDatabase.insert(DbOpenHelper.BOOK_TABLE_NAME, null, contentValues);
        contentValues.put("bookName", "编译原理");
        sqLiteDatabase.insert(DbOpenHelper.BOOK_TABLE_NAME, null, contentValues);
        contentValues.put("bookName", "网络原理");
        sqLiteDatabase.insert(DbOpenHelper.BOOK_TABLE_NAME, null, contentValues);
        contentValues.clear();

        contentValues.put("userName", "叶");
        contentValues.put("sex", "女");
        sqLiteDatabase.insert(DbOpenHelper.USER_TABLE_NAME, null, contentValues);
        contentValues.put("userName", "叶叶");
        contentValues.put("sex", "男");
        sqLiteDatabase.insert(DbOpenHelper.USER_TABLE_NAME, null, contentValues);
        contentValues.put("userName", "leavesc");
        contentValues.put("sex", "男");
        sqLiteDatabase.insert(DbOpenHelper.USER_TABLE_NAME, null, contentValues);
        sqLiteDatabase.setTransactionSuccessful();
        sqLiteDatabase.endTransaction();
    }

    @Override
    public String getType(Uri uri) {
        return null;
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("Unsupported URI:" + uri);
        }
        sqLiteDatabase.insert(tableName, null, values);
        context.getContentResolver().notifyChange(uri, null);
        return uri;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("Unsupported URI:" + uri);
        }
        return sqLiteDatabase.query(tableName, projection, selection, selectionArgs, null, null, sortOrder, null);
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("Unsupported URI:" + uri);
        }
        int row = sqLiteDatabase.update(tableName, values, selection, selectionArgs);
        if (row > 0) {
            context.getContentResolver().notifyChange(uri, null);
        }
        return row;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("Unsupported URI:" + uri);
        }
        int count = sqLiteDatabase.delete(tableName, selection, selectionArgs);
        if (count > 0) {
            context.getContentResolver().notifyChange(uri, null);
        }
        return count;
    }

}
```
注册ContentProvider

```java
        <provider
            android:name=".BookProvider"
            android:authorities="com.czy.contentproviderdemo.BookProvider" />
```
然后分别操作book和user两张表，向其插入一条数据后Log输出所有的数据
```java
public class MainActivity extends AppCompatActivity {

    private final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Uri bookUri = Uri.parse("content://com.czy.contentproviderdemo.BookProvider/book");
        ContentValues contentValues = new ContentValues();
        contentValues.put("bookName", "叫什么名字好呢");
        getContentResolver().insert(bookUri, contentValues);
        Cursor bookCursor = getContentResolver().query(bookUri, new String[]{"_id", "bookName"}, null, null, null);
        if (bookCursor != null) {
            while (bookCursor.moveToNext()) {
                Log.e(TAG, "ID:" + bookCursor.getInt(bookCursor.getColumnIndex("_id"))
                        + "  BookName:" + bookCursor.getString(bookCursor.getColumnIndex("bookName")));
            }
            bookCursor.close();
        }

        Uri userUri = Uri.parse("content://com.czy.contentproviderdemo.BookProvider/user");
        contentValues.clear();
        contentValues.put("userName", "叶叶叶");
        contentValues.put("sex", "男");
        getContentResolver().insert(userUri, contentValues);
        Cursor userCursor = getContentResolver().query(userUri, new String[]{"_id", "userName", "sex"}, null, null, null);
        if (userCursor != null) {
            while (userCursor.moveToNext()) {
                Log.e(TAG, "ID:" + userCursor.getInt(userCursor.getColumnIndex("_id"))
                        + "  UserName:" + userCursor.getString(userCursor.getColumnIndex("userName"))
                        + "  Sex:" + userCursor.getString(userCursor.getColumnIndex("sex")));
            }
            userCursor.close();
        }
    }

}
```
得到的输出结果是：

```java
04-03 16:54:34.010 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:1  BookName:数据结构
04-03 16:54:34.010 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:2  BookName:编译原理
04-03 16:54:34.010 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:3  BookName:网络原理
04-03 16:54:34.010 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:4  BookName:叫什么名字好呢
04-03 16:54:34.018 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:1  UserName:叶  Sex:女
04-03 16:54:34.018 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:2  UserName:叶叶  Sex:男
04-03 16:54:34.018 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:3  UserName:leavesc  Sex:男
04-03 16:54:34.018 25151-25151/com.czy.contentproviderdemo E/MainActivity: ID:4  UserName:叶叶叶  Sex:男
```

## **六、获取联系人信息**

获取联系人列表

```java
public void showContacts(View view) {
        ContentResolver contentResolver = getContentResolver();
        Cursor cursor = contentResolver.query(ContactsContract.Contacts.CONTENT_URI, null, null, null, null);
        if (cursor == null) {
            tv_contacts.setText("显示联系人错误");
            return;
        }
        StringBuilder stringBuilder = new StringBuilder();
        while (cursor.moveToNext()) {
            stringBuilder.append("ID：");
            String contactId = cursor.getString(cursor.getColumnIndex(ContactsContract.Contacts._ID));
            stringBuilder.append(contactId);

            stringBuilder.append("\t\t");
            stringBuilder.append("名字：");
            String contactName = cursor.getString(cursor.getColumnIndex(ContactsContract.Contacts.DISPLAY_NAME));
            stringBuilder.append(contactName);

            // 根据联系人ID查询对应的电话号码
            Cursor phonesCursor = contentResolver.query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
                    null, ContactsContract.CommonDataKinds.Phone.CONTACT_ID + " = " + contactId, null, null);
            // 取得电话号码(可能会存在多个号码)
            if (phonesCursor != null) {
                stringBuilder.append("\t\t");
                stringBuilder.append("号码：");
                while (phonesCursor.moveToNext()) {
                    String phoneNumber = phonesCursor.getString(phonesCursor.getColumnIndex(ContactsContract.CommonDataKinds.Phone.NUMBER));
                    stringBuilder.append(phoneNumber);
                    stringBuilder.append("\n");
                }
                phonesCursor.close();
            }
        }
        cursor.close();
        tv_contacts.setText(stringBuilder.toString());
    }
```
添加指定名字的联系人

```java
 public void addContact(View view) {
        ContentValues values = new ContentValues();
        //首先向RawContacts.CONTENT_URI执行一个空值插入，目的是获取系统返回的rawContactId
        Uri rawContactUri = getContentResolver().insert(ContactsContract.RawContacts.CONTENT_URI, values);
        long rawContactId = ContentUris.parseId(rawContactUri);
        //往data表入姓名数据
        values.clear();
        values.put(ContactsContract.Data.RAW_CONTACT_ID, rawContactId);
        values.put(ContactsContract.Data.MIMETYPE, ContactsContract.CommonDataKinds.StructuredName.CONTENT_ITEM_TYPE);
        values.put(ContactsContract.CommonDataKinds.StructuredName.GIVEN_NAME, "leavesc");
        getContentResolver().insert(android.provider.ContactsContract.Data.CONTENT_URI, values);
        //往data表入电话数据
        values.clear();
        values.put(android.provider.ContactsContract.Contacts.Data.RAW_CONTACT_ID, rawContactId);
        values.put(ContactsContract.Data.MIMETYPE, ContactsContract.CommonDataKinds.Phone.CONTENT_ITEM_TYPE);
        values.put(ContactsContract.CommonDataKinds.Phone.NUMBER, "19950724");
        values.put(ContactsContract.CommonDataKinds.Phone.TYPE, ContactsContract.CommonDataKinds.Phone.TYPE_MOBILE);
        getContentResolver().insert(android.provider.ContactsContract.Data.CONTENT_URI, values);
        //往data表入Email数据
        values.clear();
        values.put(android.provider.ContactsContract.Contacts.Data.RAW_CONTACT_ID, rawContactId);
        values.put(ContactsContract.Data.MIMETYPE, ContactsContract.CommonDataKinds.Email.CONTENT_ITEM_TYPE);
        values.put(ContactsContract.CommonDataKinds.Email.DATA, "qq@qq.com");
        values.put(ContactsContract.CommonDataKinds.Email.TYPE, ContactsContract.CommonDataKinds.Email.TYPE_WORK);
        getContentResolver().insert(android.provider.ContactsContract.Data.CONTENT_URI, values);
    }
```
需要添加联系人读写权限

```java
    <uses-permission android:name="android.permission.READ_CONTACTS" />
    <uses-permission android:name="android.permission.WRITE_CONTACTS" />
```
