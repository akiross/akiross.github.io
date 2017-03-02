# An introduction to LMDB & Caffe

Lightning Memory-Mapped Database, LMDB from now on, is a database supported by caffe. It is convenient to store large amount of data in the form of key-value store.

## Installation notes

I'm using LMDB from python on Fedora 25, installed using pip. I had to install `redhat-rpm-config` to make it work:

    sudo dnf install redhat-rpm-config
	pip3 install --user lmdb

Notice that I am using python3. LMDB uses strings of bytes and python3 has unicode strings, so keep an eye on it.

## Basic management
Let's start with a simple hello world database:

    import lmdb

	env = lmdb.open('my_database')

This code will build a single named database. In LMDB you have the concept of **environment**, which could actually be a number of different databases in the same file. In this case, since we did not specify any number, a single database will be used.

Let's put some data in it:

    with env.begin(write=True) as txn:
		txn.put(b'h', b'e')

`env.begin` will build a **transaction** object: LMDB is in fact a transactional database, meaning that you can organize your operations on it in a group (a transaction). If something goes wrong, you can interrupt a transaction and your database is safe. Transactions help in keeping the state of the database correct and consistent, in case something goes wrong.

In this case, we want to build a writing transaction, therefore we specify `write=True`. What we get from `begin` is a transaction object, `txn`, that we can use to write a key-value pair onto the db.

When the context manager exits, the transaction is executed and the change is stored in the database. Let's close the environment and see what we get.

    env.close()

After executing this code (which I wrote in a ipython session), in the current working directory we see the directory `my_database`, which contains two files: `data.mdb` and `lock.mdb`. The data.mdb file contains our data, and lock.mdb is used for concurrent access to the data. This is because, by default, `Environment()` constructor has the `subdir=True` keyword argument set.

Let's go back to our code and try to read the data. We are going to use a `Cursor` to iterate over the key-value pairs (currently, only one):

	with env.begin() as txn:
		for key, val in txn.cursor():
			print(key, val)

running this code we get `b'h' b'e'` as expected: the pair we inserted above. Note how we called `begin()` with no arguments for reading.
Let's inspect the key-value storage behavior:

    with env.begin(write=True) as txn:
	    hw = 'hello, world'
		with a, b in zip(hw, hw[1:]):
		    txn.put(a.encode(), b.encode())

	with env.begin() as txn:
	    for k, v in txn.cursor():
		    print(k, v)

Will output the following:

	b' ' b'w'
	b',' b' '
	b'e' b'l'
	b'h' b'e'
	b'l' b'd'
	b'o' b'r'
	b'r' b'l'
	b'w' b'o'

First, note how the keys are sorted: this is a feature of LMDB and keys are always sorted lexicographically. Your code can rely on this. Second, note that this key-value map behaves as usual: if a key already exists, `put()` will overwrite the previous value. This is a map, not a multi-map.

This latter behavior can be modified in two ways:

 1. by setting the `overwrite=False` parameter in `put()`, if the value is already present, put will not write over it;
 2. by using sub-databases (we'll get to it), it is possible to have duplicated keys (a multi-map).

Being a key-value database, random access is also present: you don't have to use a cursor to access a specific key, but you can use the `get()` method of a transaction:

    with env.begin() as txn:
	    print(txn.get(b'h')) # b'e'

It is important to know that the `get()` method has a `default=None` keyword argument: if the key does not exist, 

Other interesting methods for accessing the data are `pop()` and `replace()`:

	with env.begin(write=True) as txn:
		print(txn.get(b'h', default='miss')) # prints b'e'
		print(txn.pop(b'h')),                # prints b'e'
		print(txn.get(b'h', default='miss')) # prints 'miss'
		print(txn.replace(b'h', b'i'))       # prints None (old val)
		print(txn.get(b'h'))                 # prints b'i'

Note that we had to set `write=True` to use pop and replace.

Also, you can create a `Cursor` move in the database using its `set_key()`, `prev()` and `next()` methods, and by using `item()`, `key()` and `value()` methods to access the data:

	with env.begin(write=True) as txn:
		cur = txn.cursor()
		print(cur.key(), cur.value()) # b'' b'', cur is unpositioned
		cur.next()
		print(cur.key(), cur.value()) # b' ' b'w', first key
		cur.set_key(b'o')
		print(cur.item()) # (b'o', b'r')
		cur.prev()
		print(cur.item()) # (b'l', b'd')

## Sub-databases
Even if the API shown above are enough for the majority of applications, LMDB allows to put multiple databases into a single environment. Each sub-database is identified by a name, but careful: the name must not conflict with a key!

We will demonstrate a simple example, but we won't go into details.

    import lmdb

	env = lmdb.open('my_multi_db', max_dbs=2) # Max number of databases
	sub = env.open_db(b'my_sub_db', dupsort=True) # Allow duplicated keys

	with env.begin(write=True, db=sub):
		hw = 'hello, world!'
		for a, b in zip(hw, hw[1]):
			txn.put(a.encode(), b.encode())

	with env.begin(db=sub) as txn:
		for k, v in txn.cursor():
			print(k, v)

This code will produce the following:

	b' ' b'w'
	b',' b' '
	b'd' b'!'
	b'e' b'l'
	b'h' b'e'
	b'l' b'd'
	b'l' b'l'
	b'l' b'o'
	b'o' b','
	b'o' b'r'
	b'r' b'l'
	b'w' b'o'

Note how, to operate over the sub-database, we had to use the `db=` argument. In general, `db` is diffused in the LMDB API, and if it's not specified (`None`), it will default to the main database.

Also, note how we could enable duplicated keys, and now it's possible to map multiple values to the same key. To get the values, you can use the cursor `set_key()` as we did earlier and use `prev()` and `next()`, or you can use the other very useful functions, like `next_dup()`, `next_nodup()`, `first_dup()`, `last_dup()`, `set_key_dup()` etc.

## LMDB in Caffe

Now that LMDB has no more secrets for us (HA!), we can use it to store data in Caffe. LMDB is convenient for large datasets, not only because it's fast, but also because it manages the memory-mapping by itself, without filling your RAM.

When manipulating Caffe datasets with LMDB, the important bit is the [Datum Object](https://github.com/BVLC/caffe/wiki/The-Datum-Object). The Datum allows you to store optionally-labeled data, and allows easy conversion to bytestring, so that you can use it in LMDB.

Being a protobuf message class, you can inspect how a Datum is composed by looking at its [protobuf definition](https://github.com/BVLC/caffe/blob/master/src/caffe/proto/caffe.proto#L30) ((feel free to PR if the line changes)).

Here's how to use it:

	import lmdb
    import caffe

	dat = caffe.proto.caffe_pb2.Datum()
	dat.channels = 1
	dat.height = 3
	dat.width = 4
	dat.data = b'hello, world'
	dat.label = 42

Datum has a data field in bytes, but you can also store floats in it, but the method is rather flexible and you can [store other data types](https://github.com/BVLC/caffe/issues/2116) in it.

After filling a datum, you can write it to a database, after converting it to bytes. Luckly, protobuf supports a method to easily [convert a buffer to a string](https://developers.google.com/protocol-buffers/docs/pythontutorial):

	env = lmdb.open('my_dataset')
	with env.begin(write=True) as txn:
		txn.put(b'somekey', dat.SerializeToString())

To read the data, you can use the inverse method:

	with env.begin() as txn:
		binar = txn.get(b'somekey')
		datum = caffe.proto.caffe_pb2.Datum()
		datum.ParseFromString(binar)

And that's it!
