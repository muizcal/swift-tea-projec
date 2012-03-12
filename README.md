Swift
=====

* [source](http://github.com/shanna/swift)
* [documentation](http://rubydoc.info/gems/swift/file/README.md)

## Description

A rational rudimentary object relational mapper.

## Dependencies

* ruby   >= 1.9.1
* [dbic++](http://github.com/deepfryed/dbicpp) >= 0.6.0
* mysql  >= 5.0.17, postgresql >= 8.4 or sqlite3 >= 3.7

## Features

* Multiple databases.
* Prepared statements.
* Bind values.
* Transactions and named save points.
* Asynchronous API for PostgreSQL and MySQL.
* IdentityMap.
* Migrations.

## Synopsis

### DB

```ruby
  require 'swift'

  Swift.trace true # Debugging.
  Swift.setup :default, Swift::DB::Postgres, db: 'swift'

  # Block form db context.
  Swift.db do |db|
    db.execute('drop table if exists users')
    db.execute('create table users(id serial, name text, email text)')

    # Save points are supported.
    db.transaction :named_save_point do
      st = db.prepare('insert into users (name, email) values (?, ?) returning id')
      puts st.execute('Apple Arthurton', 'apple@arthurton.local').insert_id
      puts st.execute('Benny Arthurton', 'benny@arthurton.local').insert_id
    end

    # Block result iteration.
    db.prepare('select * from users').execute do |row|
      puts row.inspect
    end

    # Enumerable.
    result = db.prepare('select * from users where name like ?').execute('Benny%')
    puts result.first
  end
```

### DB Scheme Operations

Rudimentary object mapping. Provides a definition to the db methods for prepared (and cached) statements plus native
primitive Ruby type conversion.

```ruby
  require 'swift'
  require 'swift/migrations'

  Swift.trace true # Debugging.
  Swift.setup :default, Swift::DB::Postgres, db: 'swift'

  class User < Swift::Scheme
    store     :users
    attribute :id,         Swift::Type::Integer, serial: true, key: true
    attribute :name,       Swift::Type::String
    attribute :email,      Swift::Type::String
    attribute :updated_at, Swift::Type::Time
  end # User

  Swift.db do |db|
    db.migrate! User

    # Select Scheme instance (relation) instead of Hash.
    users = db.prepare(User, 'select * from users limit 1').execute

    # Make a change and update.
    users.each{|user| user.updated_at = Time.now}
    db.update(User, *users)

    # Get a specific user by id.
    user = db.get(User, id: 1)
    puts user.name, user.email
  end
```

### Scheme CRUD

Scheme/relation level helpers.

```ruby
  require 'swift'
  require 'swift/migrations'

  Swift.trace true # Debugging.
  Swift.setup :default, Swift::DB::Postgres, db: 'swift'

  class User < Swift::Scheme
    store     :users
    attribute :id,    Swift::Type::Integer, serial: true, key: true
    attribute :name,  Swift::Type::String
    attribute :email, Swift::Type::String
  end # User

  # Migrate it.
  User.migrate!

  # Create
  User.create name: 'Apple Arthurton', email: 'apple@arthurton.local' # => User

  # Get by key.
  user = User.get id: 1

  # Alter attribute and update in one.
  user.update name: 'Jimmy Arthurton'

  # Alter attributes and update.
  user.name = 'Apple Arthurton'
  user.update

  # Destroy
  user.delete
```

### Conditions SQL syntax.

SQL is easy and most people know it so Swift ORM provides simple #to_s
attribute to table and field name typecasting.

```ruby
  class User < Swift::Scheme
    store     :users
    attribute :id,    Swift::Type::Integer, serial: true, key: true
    attribute :age,   Swift::Type::Integer, field: 'ega'
    attribute :name,  Swift::Type::String,  field: 'eman'
    attribute :email, Swift::Type::String,  field: 'liame'
  end # User

  # Convert :name and :age to fields.
  # select * from users where eman like '%Arthurton' and ega > 20
  users = User.execute(
    %Q{select * from #{User} where #{User.name} like ? and #{User.age} > ?},
    '%Arthurton', 20
  )
```

### Identity Map

Swift comes with a simple identity map. Just require it after you load swift.

```ruby
  require 'swift'
  require 'swift/identity_map'
  require 'swift/migrations'

  class User < Swift::Scheme
    store     :users
    attribute :id,    Swift::Type::Integer, serial: true, key: true
    attribute :age,   Swift::Type::Integer, field: 'ega'
    attribute :name,  Swift::Type::String,  field: 'eman'
    attribute :email, Swift::Type::String,  field: 'liame'
  end # User

  # Migrate it.
  User.migrate!

  # Create
  User.create name: 'James Arthurton', email: 'james@arthurton.local' # => User

  find_user = User.prepare(%Q{select * from #{User} where #{User.name = ?})
  find_user.execute('James Arthurton')
  find_user.execute('James Arthurton') # Gets same object reference
```

### Bulk inserts

Swift comes with adapter level support for bulk inserts for MySQL and PostgreSQL. This
is usually very fast (~5-10x faster) than regular prepared insert statements for larger
sets of data.

MySQL adapter - Overrides the MySQL C API and implements its own _infile_ handlers. This
means currently you *cannot* execute the following SQL using Swift

```sql
  LOAD DATA LOCAL INFILE '/tmp/users.tab' INTO TABLE users;
```

But you can do it almost as fast in ruby,

```ruby
  require 'swift'

  Swift.setup :default, Swift::DB::Mysql, db: 'swift'

  # MySQL packet size is the usual limit, 8k is the packet size by default.
  Swift.db do |db|
    File.open('/tmp/users.tab') do |file|
      count = db.write('users', %w{name email balance}, file)
    end
  end
```

You are not just limited to files - you can stream data from anywhere into your database without
creating temporary files.

### Asynchronous API

Swift::Adapter#aexecute returns a Swift::Result instance. You can either poll the corresponding Swift::Adapter#fileno
and then call Swift::Result#retrieve when ready of use a block form like below which implicitly uses rb_thread_wait_fd

```ruby
  require 'swift'

  pool = 3.times.map.with_index {|n| Swift.setup n, Swift::DB::Postgres, db: 'swift' }

  Thread.new do
    pool[0].aexecute('select pg_sleep(3), 1 as query_id') {|row| p row}
  end

  Thread.new do
    pool[1].aexecute('select pg_sleep(2), 2 as query_id') {|row| p row}
  end

  Thread.new do
    pool[2].aexecute('select pg_sleep(1), 3 as query_id') {|row| p row}
  end

  Thread.list.reject {|thread| Thread.current == thread}.each(&:join)
```

```ruby
  require 'swift'
  require 'eventmachine'
  
  pool = 3.times.map.with_index {|n| Swift.setup n, Swift::DB::Postgres, db: 'swift'}
  
  module Handler
    attr_reader :result
  
    def initialize result
      @result = result
    end
  
    def notify_readable
      result.retrieve
      result.each {|row| p row }
      unbind
    end
  end
  
  EM.run do
    EM.watch(pool[0].fileno, Handler, pool[0].aexecute('select pg_sleep(3), 1 as query_id')){|c| c.notify_readable = true}
    EM.watch(pool[1].fileno, Handler, pool[1].aexecute('select pg_sleep(2), 2 as query_id')){|c| c.notify_readable = true}
    EM.watch(pool[2].fileno, Handler, pool[2].aexecute('select pg_sleep(1), 3 as query_id')){|c| c.notify_readable = true}
    EM.add_timer(4) { EM.stop }
  end
```

## Performance

Swift prefers performance when it doesn't compromise the Ruby-ish interface. It's unfair to compare Swift to DataMapper
and ActiveRecord which suffer under the weight of support for many more databases and legacy/alternative Ruby
implementations. That said obviously if Swift were slower it would be redundant so benchmark code does exist in
http://github.com/shanna/swift/tree/master/benchmarks

### Benchmarks

#### ORM

The following bechmarks were run on a machine with 4G ram, 5200rpm sata drive,
Intel Core2Duo P8700 2.53GHz and stock PostgreSQL 8.4.1.

* 10,000 rows are created once.
* All the rows are selected once.
* All the rows are selected once and updated once.
* Memory footprint(rss) shows how much memory the benchmark used with GC disabled.
  This gives an idea of total memory use and indirectly an idea of the number of
  objects allocated and the pressure on Ruby GC if it were running. When GC is enabled,
  the actual memory consumption might be much lower than the numbers below.

```
  ./simple.rb -n1 -r10000 -s ar -s dm -s sequel -s swift

  benchmark       sys     user    total  real     rss
  ar #create      1.01    7.91    8.92   11.426   406.22m
  ar #select      0.02    0.31    0.33    0.378    40.69m
  ar #update      0.88    9.64   10.52   13.908   504.93m

  dm #create      0.23    3.52    3.75    5.405   211.00m
  dm #select      0.11    1.67    1.78    1.912   114.57m
  dm #update      0.54    7.34    7.88    9.453   531.30m

  sequel #create  0.77    4.61    5.38    8.194   235.50m
  sequel #select  0.01    0.13    0.14    0.180    12.73m
  sequel #update  0.64    4.76    5.40    7.790   229.69m

  swift #create   0.13    0.66    0.79    1.463    85.77m
  swift #select   0.01    0.10    0.11    0.135     8.92m
  swift #update   0.14    0.75    0.89    1.585    59.56m

  -- bulk insert api --
  swift #write    0.00    0.10    0.10    0.180    14.79m
```


#### Adapter

The adapter level SELECT benchmarks without using ORM.

* Same dataset as above.
* All rows are selected 5 times.
* The pg benchmark uses pg_typecast gem to provide typecasting support
  for pg gem and also makes the benchmarks more fair.

##### PostgreSQL

```
  benchmark       sys       user      total     real      rss
  do #select      0.020000  1.250000  1.270000  1.441281  71.98m
  pg #select      0.000000  0.580000  0.580000  0.769186  42.93m
  swift #select   0.040000  0.510000  0.550000  0.627581  43.23m
```

##### MySQL

```
  benchmark       sys       user      total     real      rss
  do #select      0.030000  1.130000  1.160000  1.172205  71.86m
  mysql2 #select  0.040000  0.660000  0.700000  0.704414  72.72m
  swift #select   0.010000  0.480000  0.490000  0.499643  42.03m
```

## TODO

* More tests.
* Assertions for dumb stuff.
* Auto-generate schema?
* Move examples to Wiki. Examples of models built on top of Schema.

## Contributing

Go nuts! There is no style guide and I do not care if you write tests or comment code. If you write something neat just
send a pull request, tweet, email or yell it at me line by line in person.
