#########################
Handcrafting Transactions
#########################

For those who wish to assemble transaction payloads "by hand", with examples in
Python.

.. contents::
    :local:
    :depth: 1

********
Overview
********

Submitting a transaction to a BigchainDB node consists of three main steps:

1. Preparing the transaction payload;
2. Fulfilling the prepared transaction payload; and
3. Sending the transaction payload via HTTPS.

Step 1 and 2 can be performed offline on the client. That is, they do not
require any connection to any BigchainDB node.

For convenience's sake, some utilites are provided to prepare and fulfill a
transaction via the :class:`~.bigchaindb_driver.BigchainDB` class, and via the
:mod:`~bigchaindb_driver.offchain` module. For an introduction on using these
utilities, see the :ref:`basic-usage` or :ref:`advanced-usage` sections.

The rest of this document will guide you through completing steps 1 and 2
manually by revisiting some of the examples provided in the usage sections.
We will:

* provide all values, including the default ones;
* generate the transaction id;
* learn to use crypto-conditions to generate a condition that locks the
  transaction, hence protecting it from being consumed by an unauthorized user;
* learn to use crypto-conditions to generate a fulfillment that unlocks
  the transaction asset, and consequently enact an ownership transfer.

In order to perform all of the above, we'll use the following Python libraries:

* :mod:`json`: to serialize the transaction dictionary into a JSON formatted
  string;
* `sha3`_: to hash the serialized transaction; and
* `cryptoconditions`_: to create conditions and fulfillments


High-level view of a transaction in Python
==========================================
For detailled documentation on the transaction schema, please consult
`The Transaction Model`_ and `The Transaction Schema`_.

From the point of view of Python, a transaction is simply a dictionary:

.. code-block:: python

    {
        'id': '5c89494da317a343e7b99a8928669efe738aef6c1c4f092e436281ca070f4ea2',
        'operation': 'CREATE',
        'asset': {
            'data': {
                'bicycle': {
                    'manufacturer': 'bkfab',
                    'serial_number': 'abcd1234',
                }
            },
            'divisible': False,
            'refillable': False,
            'updatable': False
            'id': 'f60ce68c-1278-48d4-85dd-8051feca2ba3',
        },
        'metadata': {'planet': 'earth'},
        'conditions': [{
            'amount': 1,
            'cid': 0,
            'condition': {
                'details': {
                    'bitmask': 32,
                    'public_key': '8xeMF83SdMCm8r4GTnjCnE3LV6X6RogmqAWijBL2RFMf',
                    'signature': None,
                    'type': 'fulfillment',
                    'type_id': 4
                },
                'uri': 'cc:4:20:dkL70TkL8ut-X9QqcloujpM10MRgbvIyVPjEer_DztY:96'
            },
            'owners_after': ['8xeMF83SdMCm8r4GTnjCnE3LV6X6RogmqAWijBL2RFMf']
        }],
        'fulfillments': [{
            'fid': 0,
            'fulfillment': 'cf:4:dkL70TkL8ut-X9QqcloujpM10MRgbvIyVPjEer_DztYZ0gTuGPQ2U4u6MpxfRaSMZ0i8gYRgGJ-XuzwLOXZylPLvQM81cZ4W_K6izsLvQuHbUiTgdtV3pzSUHDHdpIYC',
            'input': None,
            'owners_before': ['8xeMF83SdMCm8r4GTnjCnE3LV6X6RogmqAWijBL2RFMf'],
        }],
        'version': 1
    }

Because a transaction must be signed before being sent, the ``id``, must be
provided by the client.

.. important:: **Implications of Signed Payloads**

    Because BigchainDB relies on cryptographic signatures, the payloads need to
    be fully prepared and signed on the client side. This prevents the
    server(s) from tempering with the provided data.

    This enhanced security puts more work on the clients, as various values
    that could traditionally be generated on the server side need to be
    generated on the client side.


.. _bicycle-asset-creation-revisited:

********************************
Bicycle Asset Creation Revisited
********************************

The Prepared Transaction
========================
Recall that in order to prepare a transaction, we had to do something similar
to:

.. ipython::

    In [0]: from bigchaindb_driver.crypto import generate_keypair

    In [0]: from bigchaindb_driver.offchain import prepare_transaction

    In [0]: alice = generate_keypair()

    In [0]: bicycle = {
       ...:     'data': {
       ...:         'bicycle': {
       ...:             'serial_number': 'abcd1234',
       ...:             'manufacturer': 'bkfab',
       ...:         },
       ...:     },
       ...: }

    In [0]: metadata = {'planet': 'earth'}

    In [0]: prepared_creation_tx = prepare_transaction(
       ...:     operation='CREATE',
       ...:     owners_before=alice.public_key,
       ...:     asset=bicycle,
       ...:     metadata=metadata,
       ...: )

and the payload of the prepared transaction looked similar to:

.. ipython::

    In [0]: prepared_creation_tx

Note ``alice``'s public key:

.. ipython::

    In [0]: alice.public_key

We are now going to craft this payload by hand.

Extract asset id:

.. ipython::

    In [0]: asset_id = prepared_creation_tx['asset']['id']

asset
-----
.. ipython::

    In [0]: asset = {
       ...:     'data': {
       ...:         'bicycle': {
       ...:             'manufacturer': 'bkfab',
       ...:             'serial_number': 'abcd1234',
       ...:         },
       ...:     },
       ...:     'divisible': False,
       ...:     'refillable': False,
       ...:     'updatable': False,
       ...:     'id': asset_id,
       ...: }

metadata
--------
.. ipython::

    In [0]: metadata = {'planet': 'earth'}

operation
---------
.. ipython::

    In [0]: operation = 'CREATE'

.. important::

    Case sensitive; all letters must be capitalized.

conditions
----------
The purpose of the condition is to lock the transaction, such that a valid
fulfillment is required to unlock it. In the case of signature-based schemes,
the lock is basically a public key, such that in order to unlock the
transaction one needs to have the private key.

Let's review the condition payload of the prepared transaction, to see what we
are aiming for:

.. ipython::

    In [0]: prepared_creation_tx['conditions'][0]

The difficult parts are the condition details and URI. We''ll now see how to
generate them using the ``cryptoconditions`` library:

.. ipython::

    In [0]: from cryptoconditions import Ed25519Fulfillment

    In [0]: ed25519 = Ed25519Fulfillment(public_key=alice.public_key)

generate the condition uri:

.. ipython::

    In [0]: ed25519.condition_uri

So now you have a condition URI for Alice's public key.

As for the details:

.. ipython::

    In [0]: ed25519.to_dict()

We can now easily assemble the ``dict`` for the condition:

.. ipython::

    In [0]: condition = {
       ...:     'amount': 1,
       ...:     'cid': 0,
       ...:     'condition': {
       ...:         'details': ed25519.to_dict(),
       ...:         'uri': ed25519.condition_uri,
       ...:     },
       ...:     'owners_after': (alice.public_key,),
       ...: }

Let's recap and set the ``conditions`` key:

.. ipython::

    In [0]: from cryptoconditions import Ed25519Fulfillment

    In [0]: ed25519 = Ed25519Fulfillment(public_key=alice.public_key)

    In [0]: condition = {
       ...:     'amount': 1,
       ...:     'cid': 0,
       ...:     'condition': {
       ...:         'details': ed25519.to_dict(),
       ...:         'uri': ed25519.condition_uri,
       ...:     },
       ...:     'owners_after': (alice.public_key,),
       ...: }

    In [0]: conditions = (condition,)

The key part is the condition URI:

.. ipython::

    In [0]: ed25519.condition_uri

To know more about its meaning, you may read the `cryptoconditions internet
draft`_.


fulfillments
------------
The fulfillment for a ``CREATE`` operation is somewhat special:

.. ipython::

    In [0]: fulfillment = {
       ...:     'fid': 0,
       ...:     'fulfillment': None,
       ...:     'input': None,
       ...:     'owners_before': (alice.public_key,)
       ...: }

* The input field is empty because it's a ``CREATE`` operation;
* The ``'fulfillemnt'`` value is ``None`` as it will be set during the
  fulfillment step; and
* The ``'owners_before'`` field identifies the issuer(s) of the asset that is
  being created.


The ``fulfillments`` value is simply a list or tuple of all fulfillments:

.. ipython::

    In [0]: fulfillments = (fulfillment,)


.. note:: You may rightfully observe that the ``prepared_creation_tx``
    fulfillment generated via the ``prepare_transaction`` function  differs:

    .. ipython::

        In [0]: prepared_creation_tx['fulfillments'][0]

    More precisely, the value of ``'fulfillment'``:

    .. ipython::

        In [0]: prepared_creation_tx['fulfillments'][0]['fulfillment']

    The quick answer is that it simply is not needed, and can be set to
    ``None``.

Putting it all together:

.. ipython::

    In [0]: handcrafted_creation_tx = {
       ...:     'asset': asset,
       ...:     'metadata': metadata,
       ...:     'operation': operation,
       ...:     'conditions': conditions,
       ...:     'fulfillments': fulfillments,
       ...:     'version': 1,
       ...: }

    In [0]: handcrafted_creation_tx

We're missing the ``id``, and we'll generate it soon, but before that, let's
recap how we've put all the code together to generate the above payload:

.. code-block:: python

    from cryptoconditions import Ed25519Fulfillment
    from bigchaindb_driver.crypto import CryptoKeypair

    alice = CryptoKeypair(
        public_key=alice.public_key,
        private_key=alice.private_key,
    )

    operation = 'CREATE'

    asset = {
        'data': {
            'bicycle': {
                'manufacturer': 'bkfab',
                'serial_number': 'abcd1234',
            },
        },
        'divisible': False,
        'refillable': False,
        'updatable': False,
        'id': asset_id,
    }

    metadata = {'planet': 'earth'}

    ed25519 = Ed25519Fulfillment(public_key=alice.public_key)

    condition = {
        'amount': 1,
        'cid': 0,
        'condition': {
            'details': ed25519.to_dict(),
            'uri': ed25519.condition_uri,
        },
        'owners_after': (alice.public_key,),
    }
    conditions = (condition,)

    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': None,
        'owners_before': (alice.public_key,)
    }
    fulfillments = (fulfillment,)

    handcrafted_creation_tx = {
        'asset': asset,
        'metadata': metadata,
        'operation': operation,
        'conditions': conditions,
        'fulfillments': fulfillments,
        'version': 1,
    }

id
--

.. ipython::

    In [0]: import json

    In [0]: from sha3 import sha3_256

    In [0]: json_str_tx = json.dumps(
       ...:     handcrafted_creation_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: txid = sha3_256(json_str_tx.encode()).hexdigest()

    In [0]: handcrafted_creation_tx['id'] = txid

Compare this to the txid of the transaction generated via
``prepare_transaction()``:

.. ipython::

    In [0]: txid == prepared_creation_tx['id']

You may observe that

.. ipython::

    In [0]: handcrafted_creation_tx == prepared_creation_tx

.. ipython::

    In [0]: from copy import deepcopy

    In [0]: # back up

    In [0]: prepared_creation_tx_bk = deepcopy(prepared_creation_tx)

    In [0]: # set fulfillment to None

    In [0]: prepared_creation_tx['fulfillments'][0]['fulfillment'] = None

    In [0]: handcrafted_creation_tx == prepared_creation_tx

Are still not equal because we used tuples instead of lists.

.. ipython::

    In [0]: # serialize to json str

    In [0]: json_str_handcrafted_tx = json.dumps(handcrafted_creation_tx, sort_keys=True)

    In [0]: json_str_prepared_tx = json.dumps(prepared_creation_tx, sort_keys=True)

.. ipython::

    In [0]: json_str_handcrafted_tx == json_str_prepared_tx

    In [0]: prepared_creation_tx = prepared_creation_tx_bk

The full handcrafted yet-to-be-fulfilled transaction payload:

.. ipython::

    In [0]: handcrafted_creation_tx


The Fulfilled Transaction
=========================

.. ipython::

    In [0]: from cryptoconditions.crypto import Ed25519SigningKey

    In [0]: from bigchaindb_driver.offchain import fulfill_transaction

    In [0]: fulfilled_creation_tx = fulfill_transaction(
       ...:     prepared_creation_tx,
       ...:     private_keys=alice.private_key,
       ...: )

    In [0]: sk = Ed25519SigningKey(alice.private_key)

    In [0]: message = json.dumps(
       ...:     handcrafted_creation_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: ed25519.sign(message.encode(), sk)

    In [0]: fulfillment = ed25519.serialize_uri()

    In [0]: handcrafted_creation_tx['fulfillments'][0]['fulfillment'] = fulfillment

Let's check this:

.. ipython::

    In [0]: fulfilled_creation_tx['fulfillments'][0]['fulfillment'] == fulfillment

    In [0]: json.dumps(fulfilled_creation_tx, sort_keys=True) == json.dumps(handcrafted_creation_tx, sort_keys=True)


In a nutshell
=============

Handcrafting a ``'CREATE'`` transaction can be done as follows:

.. code-block:: python

    import json
    from uuid import uuid4

    import sha3
    import cryptoconditions

    from bigchaindb_driver.crypto import generate_keypair


    alice = generate_keypair()

    operation = 'CREATE'

    asset_id = str(uuid4())
    asset = {
        'data': {
            'bicycle': {
                'manufacturer': 'bkfab',
                'serial_number': 'abcd1234',
            },
        },
        'divisible': False,
        'refillable': False,
        'updatable': False,
        'id': asset_id,
    }

    metadata = {'planet': 'earth'}

    ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=alice.public_key)

    condition = {
        'amount': 1,
        'cid': 0,
        'condition': {
            'details': ed25519.to_dict(),
            'uri': ed25519.condition_uri,
        },
        'owners_after': (alice.public_key,),
    }
    conditions = (condition,)

    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': None,
        'owners_before': (alice.public_key,)
    }
    fulfillments = (fulfillment,)

    handcrafted_creation_tx = {
        'asset': asset,
        'metadata': metadata,
        'operation': operation,
        'conditions': conditions,
        'fulfillments': fulfillments,
        'version': 1,
    }

    json_str_tx = json.dumps(
        handcrafted_creation_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    creation_txid = sha3.sha3_256(json_str_tx.encode()).hexdigest()

    handcrafted_creation_tx['id'] = creation_txid

    sk = cryptoconditions.crypto.Ed25519SigningKey(alice.private_key)

    message = json.dumps(
        handcrafted_creation_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    ed25519.sign(message.encode(), sk)

    fulfillment = ed25519.serialize_uri()

    handcrafted_creation_tx['fulfillments'][0]['fulfillment'] = fulfillment

Sending it over to a BigchainDB node:

.. code-block:: python

    from bigchaindb_driver import BigchainDB

    bdb = BigchainDB('http://bdb-server:9984/api/v1')
    returned_creation_tx = bdb.transactions.send(handcrafted_creation_tx)

A few checks:

.. code-block:: python

    >>> json.dumps(returned_creation_tx, sort_keys=True) == json.dumps(handcrafted_creation_tx, sort_keys=True)
    True

.. code-block:: python

    >>> bdb.transactions.status(creation_txid)
    {'status': 'valid'}

.. tip:: When checking for the status of a transaction, one should keep in
    mind tiny delays before a transaction reaches a valid status.


.. _bicycle-asset-transfer-revisited:

********************************
Bicycle Asset Transfer Revisited
********************************
In the :ref:`bicycle transfer example <bicycle-transfer>` , we showed that the
transfer transaction was prepared and fulfilled as follows:

.. ipython::

    In [0]: creation_tx = fulfilled_creation_tx

    In [0]: bob = generate_keypair()

    In [0]: cid = 0

    In [0]: condition = creation_tx['conditions'][cid]

    In [0]: transfer_input = {
       ...:     'fulfillment': condition['condition']['details'],
       ...:     'input': {
       ...:          'cid': cid,
       ...:          'txid': creation_tx['id'],
       ...:      },
       ...:      'owners_before': condition['owners_after'],
       ...: }

    In [0]: prepared_transfer_tx = prepare_transaction(
       ...:     operation='TRANSFER',
       ...:     asset=creation_tx['asset'],
       ...:     inputs=transfer_input,
       ...:     owners_after=bob.public_key,
       ...: )

    In [0]: fulfilled_transfer_tx = fulfill_transaction(
       ...:     prepared_transfer_tx,
       ...:     private_keys=alice.private_key,
       ...: )

    In [0]: fulfilled_transfer_tx

Our goal is now to handcraft a payload equal to ``fulfilled_transfer_tx`` with
the help of

* :mod:`json`: to serialize the transaction dictionary into a JSON formatted
  string.
* `sha3`_: to hash the serialized transaction
* `cryptoconditions`_: to create conditions and fulfillments

The Prepared Transaction
========================

asset
-----

.. ipython::

    In [0]: asset = {'id': asset_id}

metadata
--------
.. ipython::

    In [0]: metadata = None

operation
---------
.. ipython::

    In [0]: operation = 'TRANSFER'

conditions
----------
.. ipython::

    In [0]: from cryptoconditions import Ed25519Fulfillment

    In [0]: ed25519 = Ed25519Fulfillment(public_key=bob.public_key)

    In [0]: condition = {
       ...:     'amount': 1,
       ...:     'cid': 0,
       ...:     'condition': {
       ...:         'details': ed25519.to_dict(),
       ...:         'uri': ed25519.condition_uri,
       ...:     },
       ...:     'owners_after': (bob.public_key,),
       ...: }

    In [0]: conditions = (condition,)

fulfillments
------------
.. ipython::

    In [0]: fulfillment = {
       ...:     'fid': 0,
       ...:     'fulfillment': None,
       ...:     'input': {
       ...:         'txid': creation_tx['id'],
       ...:         'cid': 0,
       ...:     },
       ...:     'owners_before': (alice.public_key,)
       ...: }

    In [0]: fulfillments = (fulfillment,)

A few notes:

* The ``input`` field points to the condition that needs to be fulfilled;
* The ``'fulfillment'`` value is ``None`` as it will be set during the
  fulfillment step; and
* The ``'owners_before'`` field identifies the fulfiller(s).

Putting it all together:

.. ipython::

    In [0]: handcrafted_transfer_tx = {
       ...:     'asset': asset,
       ...:     'metadata': metadata,
       ...:     'operation': operation,
       ...:     'conditions': conditions,
       ...:     'fulfillments': fulfillments,
       ...:     'version': 1,
       ...: }

    In [0]: handcrafted_transfer_tx

We're missing the ``id``, and we'll generate it, but before, let's recap how
we've put all the code together to generate the above payload:

.. code-block:: python

    from cryptoconditions import Ed25519Fulfillment
    from bigchaindb_driver.crypto import CryptoKeypair

    bob = CryptoKeypair(
        public_key=bob.public_key,
        private_key=bob.private_key,
    )

    operation = 'TRANSFER'
    asset = {'id': asset_id}
    metadata = None

    ed25519 = Ed25519Fulfillment(public_key=bob.public_key)

    condition = {
        'amount': 1,
        'cid': 0,
        'condition': {
            'details': ed25519.to_dict(),
            'uri': ed25519.condition_uri,
        },
        'owners_after': (bob.public_key,),
    }
    conditions = (condition,)

    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': {
            'txid': creation_tx['id'],
            'cid': 0,
        },
        'owners_before': (alice.public_key,)
    }
    fulfillments = (fulfillment,)

    handcrafted_transfer_tx = {
        'asset': asset,
        'metadata': metadata,
        'operation': operation,
        'conditions': conditions,
        'fulfillments': fulfillments,
        'version': 1,
    }

id
--

.. ipython::

    In [0]: import json

    In [0]: from sha3 import sha3_256

    In [0]: json_str_tx = json.dumps(
       ...:     handcrafted_transfer_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: txid = sha3_256(json_str_tx.encode()).hexdigest()

    In [0]: handcrafted_transfer_tx['id'] = txid

Compare this to the txid of the transaction generated via
``prepare_transaction()``

.. ipython::

    In [0]: txid == prepared_transfer_tx['id']

You may observe that

.. ipython::

    In [0]: handcrafted_transfer_tx == prepared_transfer_tx

.. ipython::

    In [0]: from copy import deepcopy

    In [0]: # back up

    In [0]: prepared_transfer_tx_bk = deepcopy(prepared_transfer_tx)

    In [0]: # set fulfillment to None

    In [0]: prepared_transfer_tx['fulfillments'][0]['fulfillment'] = None

    In [0]: handcrafted_transfer_tx == prepared_transfer_tx

Are still not equal because we used tuples instead of lists.

.. ipython::

    In [0]: # serialize to json str

    In [0]: json_str_handcrafted_tx = json.dumps(handcrafted_transfer_tx, sort_keys=True)

    In [0]: json_str_prepared_tx = json.dumps(prepared_transfer_tx, sort_keys=True)

.. ipython::

    In [0]: json_str_handcrafted_tx == json_str_prepared_tx

    In [0]: prepared_transfer_tx = prepared_transfer_tx_bk

The full handcrafted yet-to-be-fulfilled transaction payload:

.. ipython::

    In [0]: handcrafted_transfer_tx


The Fulfilled Transaction
=========================

.. ipython::

    In [0]: from cryptoconditions.crypto import Ed25519SigningKey

    In [0]: from bigchaindb_driver.offchain import fulfill_transaction

    In [0]: fulfilled_transfer_tx = fulfill_transaction(
       ...:     prepared_transfer_tx,
       ...:     private_keys=alice.private_key,
       ...: )

    In [0]: sk = Ed25519SigningKey(alice.private_key)

    In [0]: message = json.dumps(
       ...:     handcrafted_transfer_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: ed25519.sign(message.encode(), sk)

    In [0]: fulfillment = ed25519.serialize_uri()

    In [0]: handcrafted_transfer_tx['fulfillments'][0]['fulfillment'] = fulfillment

Let's check this:

.. ipython::

    In [0]: fulfilled_transfer_tx['fulfillments'][0]['fulfillment'] == fulfillment

    In [0]: json.dumps(fulfilled_transfer_tx, sort_keys=True) == json.dumps(handcrafted_transfer_tx, sort_keys=True)


In a nutshell
=============

.. code-block:: python

    import json

    import sha3
    import cryptoconditions

    from bigchaindb_driver.crypto import generate_keypair


    bob = generate_keypair()

    operation = 'TRANSFER'
    asset = {'id': asset_id}
    metadata = None

    ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=bob.public_key)

    condition = {
        'amount': 1,
        'cid': 0,
        'condition': {
            'details': ed25519.to_dict(),
            'uri': ed25519.condition_uri,
        },
        'owners_after': (bob.public_key,),
    }
    conditions = (condition,)

    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': {
            'txid': creation_txid,
            'cid': 0,
        },
        'owners_before': (alice.public_key,)
    }
    fulfillments = (fulfillment,)

    handcrafted_transfer_tx = {
        'asset': asset,
        'metadata': metadata,
        'operation': operation,
        'conditions': conditions,
        'fulfillments': fulfillments,
        'version': 1,
    }

    json_str_tx = json.dumps(
        handcrafted_transfer_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    transfer_txid = sha3.sha3_256(json_str_tx.encode()).hexdigest()

    handcrafted_transfer_tx['id'] = transfer_txid

    sk = cryptoconditions.crypto.Ed25519SigningKey(alice.private_key)

    message = json.dumps(
        handcrafted_transfer_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    ed25519.sign(message.encode(), sk)

    fulfillment = ed25519.serialize_uri()

    handcrafted_transfer_tx['fulfillments'][0]['fulfillment'] = fulfillment

Sending it over to a BigchainDB node:

.. code-block:: python

    from bigchaindb_driver import BigchainDB

    bdb = BigchainDB('http://bdb-server:9984/api/v1')
    returned_transfer_tx = bdb.transactions.send(handcrafted_transfer_tx)

A few checks:

.. code-block:: python

    >>> json.dumps(returned_transfer_tx, sort_keys=True) == json.dumps(handcrafted_transfer_tx, sort_keys=True)
    True

.. code-block:: python

    >>> bdb.transactions.status(transfer_txid)
    {'status': 'valid'}

.. tip:: When checking for the status of a transaction, one should keep in
    mind tiny delays before a transaction reaches a valid status.


*************************
Bicycle Sharing Revisited
*************************

Handcrafting the ``'CREATE'`` transaction:

.. code-block:: python

    import json
    from uuid import uuid4

    import sha3
    import cryptoconditions

    from bigchaindb_driver.crypto import generate_keypair


    bob, carly = generate_keypair(), generate_keypair()

    asset_id = str(uuid4())
    asset = {
        'divisible': True,
        'data': {
            'token_for': {
                'bicycle': {
                    'manufacturer': 'bkfab',
                    'serial_number': 'abcd1234',
                },
                'description': 'time share token. each token equals 1 hour of riding.'
            },
        },
        'refillable': False,
        'updatable': False,
        'id': asset_id,
    }

    # CRYPTO-CONDITIONS: instantiate an Ed25519 crypto-condition for carly
    ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=carly.public_key)

    # CRYPTO-CONDITIONS: generate the condition uri
    condition_uri = ed25519.condition.serialize_uri()

    # CRYPTO-CONDITIONS: get the unsigned fulfillment dictionary (details)
    unsigned_fulfillment_dict = ed25519.to_dict()

    condition = {
        'amount': 10,
        'cid': 0,
        'condition': {
            'details': unsigned_fulfillment_dict,
            'uri': condition_uri,
        },
        'owners_after': (carly.public_key,),
    }

    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': None,
        'owners_before': (bob.public_key,)
    }

    token_creation_tx = {
        'asset': metadata': None,
        'operation': 'CREATE',
        'conditions': (condition,),
        'fulfillments': (fulfillment,),
        'version': 1,
    }

    # JSON: serialize the id-less transaction to a json formatted string
    json_str_tx = json.dumps(
        token_creation_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    # SHA3: hash the serialized id-less transaction to generate the id
    creation_txid = sha3.sha3_256(json_str_tx.encode()).hexdigest()

    # add the id
    token_creation_tx['id'] = creation_txid

    # JSON: serialize the transaction-with-id to a json formatted string
    message = json.dumps(
        token_creation_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    # CRYPTO-CONDITIONS: sign the serialized transaction-with-id
    ed25519.sign(message.encode(),
                 cryptoconditions.crypto.Ed25519SigningKey(bob.private_key))

    # CRYPTO-CONDITIONS: generate the fulfillment uri
    fulfillment_uri = ed25519.serialize_uri()

    # add the fulfillment uri (signature)
    token_creation_tx['fulfillments'][0]['fulfillment'] = fulfillment_uri

Sending it over to a BigchainDB node:

.. code-block:: python

    from bigchaindb_driver import BigchainDB

    bdb = BigchainDB('http://bdb-server:9984/api/v1')
    returned_creation_tx = bdb.transactions.send(token_creation_tx)

A few checks:

.. code-block:: python

    >>> json.dumps(returned_creation_tx, sort_keys=True) == json.dumps(token_creation_tx, sort_keys=True)
    True

    >>> token_creation_tx['fulfillments'][0]['owners_before'][0] == bob.public_key
    True

    >>> token_creation_tx['conditions'][0]['owners_after'][0] == carly.public_key
    True

    >>> token_creation_tx['conditions'][0]['amount'] == 10
    True


.. code-block:: python

    >>> bdb.transactions.status(creation_txid)
    {'status': 'valid'}

.. tip:: When checking for the status of a transaction, one should keep in
    mind tiny delays before a transaction reaches a valid status.


Now Carly wants to ride the bicycle for 2 hours so she needs to send 2 tokens
to Bob:

.. code-block:: python

    # CRYPTO-CONDITIONS: instantiate an Ed25519 crypto-condition for carly
    bob_ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=bob.public_key)

    # CRYPTO-CONDITIONS: instantiate an Ed25519 crypto-condition for carly
    carly_ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=carly.public_key)

    # CRYPTO-CONDITIONS: generate the condition uris
    bob_condition_uri = bob_ed25519.condition.serialize_uri()
    carly_condition_uri = carly_ed25519.condition.serialize_uri()

    # CRYPTO-CONDITIONS: get the unsigned fulfillment dictionary (details)
    bob_unsigned_fulfillment_dict = bob_ed25519.to_dict()
    carly_unsigned_fulfillment_dict = carly_ed25519.to_dict()

    bob_condition = {
        'amount': 2,
        'cid': 0,
        'condition': {
            'details': bob_unsigned_fulfillment_dict,
            'uri': bob_condition_uri,
        },
        'owners_after': (bob.public_key,),
    }
    carly_condition = {
        'amount': 8,
        'cid': 1,
        'condition': {
            'details': carly_unsigned_fulfillment_dict,
            'uri': carly_condition_uri,
        },
        'owners_after': (carly.public_key,),
    }

    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': {
            'txid': token_creation_tx['id'],
            'cid': 0,
        },
        'owners_before': (carly.public_key,)
    }

    token_transfer_tx = {
        'asset': {'id': asset_id},
        'metadata': None,
        'operation': 'TRANSFER',
        'conditions': (bob_condition, carly_condition),
        'fulfillments': (fulfillment,),
        'version': 1,
    }

    # JSON: serialize the id-less transaction to a json formatted string
    json_str_tx = json.dumps(
        token_transfer_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    # SHA3: hash the serialized id-less transaction to generate the id
    transfer_txid = sha3.sha3_256(json_str_tx.encode()).hexdigest()

    # add the id
    token_transfer_tx['id'] = transfer_txid

    # JSON: serialize the transaction-with-id to a json formatted string
    message = json.dumps(
        token_transfer_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    # CRYPTO-CONDITIONS: sign the serialized transaction-with-id for bob
    carly_ed25519.sign(message.encode(),
                     cryptoconditions.crypto.Ed25519SigningKey(carly.private_key))

    # CRYPTO-CONDITIONS: generate bob's fulfillment uri
    fulfillment_uri = carly_ed25519.serialize_uri()

    # add bob's fulfillment uri (signature)
    token_transfer_tx['fulfillments'][0]['fulfillment'] = fulfillment_uri

Sending it over to a BigchainDB node:

.. code-block:: python

    bdb = BigchainDB('http://bdb-server:9984/api/v1')
    returned_transfer_tx = bdb.transactions.send(token_transfer_tx)

A few checks:

.. code-block:: python

    >>> json.dumps(returned_transfer_tx, sort_keys=True) == json.dumps(token_transfer_tx, sort_keys=True)
    True

    >>> token_transfer_tx['fulfillments'][0]['owners_before'][0] == carly.public_key
    True


.. code-block:: python

    >>> bdb.transactions.status(creation_txid)
    {'status': 'valid'}

.. tip:: When checking for the status of a transaction, one should keep in
    mind tiny delays before a transaction reaches a valid status.

*************************
Multiple Owners Revisited
*************************

Walkthrough
===========

We'll re-use the example, to compare our work.

Say ``alice`` and ``bob`` own a car together:

.. ipython::

    In [0]: from bigchaindb_driver.crypto import generate_keypair

    In [0]: from bigchaindb_driver import offchain

    In [0]: alice, bob = generate_keypair(), generate_keypair()

    In [0]: car_asset = {'data': {'car': {'vin': '5YJRE11B781000196'}}}

    In [0]: car_creation_tx = offchain.prepare_transaction(
       ...:     operation='CREATE',
       ...:     owners_before=alice.public_key,
       ...:     owners_after=(alice.public_key, bob.public_key),
       ...:     asset=car_asset,
       ...: )

    In [0]: signed_car_creation_tx = offchain.fulfill_transaction(
       ...:     car_creation_tx,
       ...:     private_keys=alice.private_key,
       ...: )

    In [0]: signed_car_creation_tx


.. code-block:: python

    sent_car_tx = bdb.transactions.send(signed_car_creation_tx)

One day, ``alice`` and ``bob``, having figured out how to teleport themselves,
and realizing they no longer need their car, wish to transfer the ownership of
their car over to ``carol``:

.. ipython::

    In [0]: carol = generate_keypair()

    In [0]: cid = 0

    In [0]: condition = signed_car_creation_tx['conditions'][cid]

    In [0]: input_ = {
       ...:     'fulfillment': condition['condition']['details'],
       ...:     'input': {
       ...:         'cid': cid,
       ...:         'txid': signed_car_creation_tx['id'],
       ...:     },
       ...:     'owners_before': condition['owners_after'],
       ...: }

    In [0]: asset = signed_car_creation_tx['asset']

    In [0]: car_transfer_tx = offchain.prepare_transaction(
       ...:     operation='TRANSFER',
       ...:     owners_after=carol.public_key,
       ...:     asset=asset,
       ...:     inputs=input_,
       ...: )

    In [0]: signed_car_transfer_tx = offchain.fulfill_transaction(
       ...:     car_transfer_tx, private_keys=[alice.private_key, bob.private_key]
       ...: )

    In [0]: signed_car_transfer_tx

Sending the transaction to a BigchainDB node:

.. code-block:: python

    sent_car_transfer_tx = bdb.transactions.send(signed_car_transfer_tx)

In order to do this manually, let's first import the necessary tools (json,
sha3, and cryptoconditions):

.. ipython::

    In [0]: import json

    In [0]: from sha3 import sha3_256

    In [0]: from cryptoconditions import Ed25519Fulfillment, ThresholdSha256Fulfillment

    In [0]: from cryptoconditions.crypto import Ed25519SigningKey

Create the asset, setting all values:

.. ipython::

    In [0]: car_asset_id = signed_car_creation_tx['asset']['id']

    In [0]: car_asset = {
       ...:     'data': {'car': {'vin': '5YJRE11B781000196'}},
       ...:     'divisible': False,
       ...:     'refillable': False,
       ...:     'updatable': False,
       ...:     'id': car_asset_id,
       ...: }

Generate the condition:

.. ipython::

    In [0]: alice_ed25519 = Ed25519Fulfillment(public_key=alice.public_key)

    In [0]: bob_ed25519 = Ed25519Fulfillment(public_key=bob.public_key)

    In [0]: threshold_sha256 = ThresholdSha256Fulfillment(threshold=2)

    In [0]: threshold_sha256.add_subfulfillment(alice_ed25519)

    In [0]: threshold_sha256.add_subfulfillment(bob_ed25519)

    In [0]: unsigned_subfulfillments_dict = threshold_sha256.to_dict()

    In [0]: condition_uri = threshold_sha256.condition.serialize_uri()

    In [0]: condition = {
       ...:     'amount': 1,
       ...:     'cid': 0,
       ...:     'condition': {
       ...:         'details': unsigned_subfulfillments_dict,
       ...:         'uri': condition_uri,
       ...:     },
       ...:     'owners_after': (alice.public_key, bob.public_key),
       ...: }

.. tip:: The condition ``uri`` could have been generated in a slightly
    different way, which may be more intuitive to you. You can think of the
    threshold condition containing sub conditions:

    .. ipython::

        In [0]: alt_threshold_sha256 = ThresholdSha256Fulfillment(threshold=2)

        In [0]: alt_threshold_sha256.add_subcondition(alice_ed25519.condition)

        In [0]: alt_threshold_sha256.add_subcondition(bob_ed25519.condition)

        In [0]: alt_threshold_sha256.condition.serialize_uri() == condition_uri

    The ``details`` on the other hand holds the associated fulfillments not yet
    fulfilled.

The yet to be fulfilled fulfillment:

.. ipython::

    In [0]: fulfillment = {
       ...:     'fid': 0,
       ...:     'fulfillment': None,
       ...:     'input': None,
       ...:     'owners_before': (alice.public_key,),
       ...: }

Craft the payload:

.. ipython::

    In [0]: handcrafted_car_creation_tx = {
       ...:     'asset': car_asset,
       ...:     'metadata': None,
       ...:     'operation': 'CREATE',
       ...:     'conditions': (condition,),
       ...:     'fulfillments': (fulfillment,),
       ...:     'version': 1,
       ...: }

Generate the id, by hashing the encoded json formatted string representation of
the transaction:

.. ipython::

    In [0]: json_str_tx = json.dumps(
       ...:     handcrafted_car_creation_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: car_creation_txid = sha3_256(json_str_tx.encode()).hexdigest()

    In [0]: handcrafted_car_creation_tx['id'] = car_creation_txid

Let's make sure our txid is the same as the one provided by the driver:

.. ipython::

    In [0]: handcrafted_car_creation_tx['id'] == car_creation_tx['id']

Sign the transaction:

.. ipython::

    In [0]: message = json.dumps(
       ...:     handcrafted_car_creation_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: alice_ed25519.sign(message.encode(), Ed25519SigningKey(alice.private_key))

    In [0]: fulfillment_uri = alice_ed25519.serialize_uri()

    In [0]: handcrafted_car_creation_tx['fulfillments'][0]['fulfillment'] = fulfillment_uri

Compare our signed CREATE transaction with the driver's:

.. ipython::

    In [0]: (json.dumps(handcrafted_car_creation_tx, sort_keys=True) ==
       ...:  json.dumps(signed_car_creation_tx, sort_keys=True))

The transfer:

.. ipython::

    In [0]: alice_ed25519 = Ed25519Fulfillment(public_key=alice.public_key)

    In [0]: bob_ed25519 = Ed25519Fulfillment(public_key=bob.public_key)

    In [0]: carol_ed25519 = Ed25519Fulfillment(public_key=carol.public_key)

    In [0]: unsigned_fulfillments_dict = carol_ed25519.to_dict()

    In [0]: condition_uri = carol_ed25519.condition.serialize_uri()

    In [0]: condition = {
       ...:     'amount': 1,
       ...:     'cid': 0,
       ...:     'condition': {
       ...:         'details': unsigned_fulfillments_dict,
       ...:         'uri': condition_uri,
       ...:     },
       ...:     'owners_after': (carol.public_key,),
       ...: }

The yet to be fulfilled fulfillments:

.. ipython::

    In [0]: fulfillment = {
       ...:     'fid': 0,
       ...:     'fulfillment': None,
       ...:     'input': {
       ...:         'txid': handcrafted_car_creation_tx['id'],
       ...:         'cid': 0,
       ...:     },
       ...:     'owners_before': (alice.public_key, bob.public_key),
       ...: }

Craft the payload:

.. ipython::

    In [0]: handcrafted_car_transfer_tx = {
       ...:     'asset': {'id': car_asset_id},
       ...:     'metadata': None,
       ...:     'operation': 'TRANSFER',
       ...:     'conditions': (condition,),
       ...:     'fulfillments': (fulfillment,),
       ...:     'version': 1,
       ...: }

Generate the id, by hashing the encoded json formatted string representation of
the transaction:

.. ipython::

    In [0]: json_str_tx = json.dumps(
       ...:     handcrafted_car_transfer_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: car_transfer_txid = sha3_256(json_str_tx.encode()).hexdigest()

    In [0]: handcrafted_car_transfer_tx['id'] = car_transfer_txid

Let's make sure our txid is the same as the one provided by the driver:

.. ipython::

    In [0]: handcrafted_car_transfer_tx['id'] == car_transfer_tx['id']

Sign the transaction:

.. ipython::

    In [0]: message = json.dumps(
       ...:     handcrafted_car_transfer_tx,
       ...:     sort_keys=True,
       ...:     separators=(',', ':'),
       ...:     ensure_ascii=False,
       ...: )

    In [0]: alice_sk = Ed25519SigningKey(alice.private_key)

    In [0]: bob_sk = Ed25519SigningKey(bob.private_key)

    In [0]: threshold_sha256 = ThresholdSha256Fulfillment(threshold=2)

    In [0]: threshold_sha256.add_subfulfillment(alice_ed25519)

    In [0]: threshold_sha256.add_subfulfillment(bob_ed25519)

    In [102]: alice_condition = threshold_sha256.get_subcondition_from_vk(alice.public_key)[0]

    In [103]: bob_condition = threshold_sha256.get_subcondition_from_vk(bob.public_key)[0]

    In [106]: alice_condition.sign(message.encode(), private_key=alice_sk)

    In [107]: bob_condition.sign(message.encode(), private_key=bob_sk)

    In [0]: fulfillment_uri = threshold_sha256.serialize_uri()

    In [0]: handcrafted_car_transfer_tx['fulfillments'][0]['fulfillment'] = fulfillment_uri

Compare our signed TRANSFER transaction with the driver's:

.. ipython::

    In [0]: (json.dumps(handcrafted_car_transfer_tx, sort_keys=True) ==
       ...:  json.dumps(signed_car_transfer_tx, sort_keys=True))

In a nutshell
=============

Handcrafting the ``'CREATE'`` transaction
-----------------------------------------

.. code-block:: python

    import json

    import sha3
    import cryptoconditions

    from bigchaindb_driver.crypto import generate_keypair


    car_asset = {
        'data': {
            'car': {
                'vin': '5YJRE11B781000196',
            },
        },
        'divisible': False,
         'refillable': False,
         'updatable': False,
         'id': '5YJRE11B781000196',
    }

    alice, bob = generate_keypair(), generate_keypair()

    # CRYPTO-CONDITIONS: instantiate an Ed25519 crypto-condition for alice
    alice_ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=alice.public_key)

    # CRYPTO-CONDITIONS: instantiate an Ed25519 crypto-condition for bob
    bob_ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=bob.public_key)

    # CRYPTO-CONDITIONS: instantiate a threshold SHA 256 crypto-condition
    threshold_sha256 = cryptoconditions.ThresholdSha256Fulfillment(threshold=2)

    # CRYPTO-CONDITIONS: add alice ed25519 to the threshold SHA 256 condition
    threshold_sha256.add_subfulfillment(alice_ed25519)

    # CRYPTO-CONDITIONS: add bob ed25519 to the threshold SHA 256 condition
    threshold_sha256.add_subfulfillment(bob_ed25519)

    # CRYPTO-CONDITIONS: get the unsigned fulfillment dictionary (details)
    unsigned_subfulfillments_dict = threshold_sha256.to_dict()

    # CRYPTO-CONDITIONS: generate the condition uri
    condition_uri = threshold_sha256.condition.serialize_uri()

    condition = {
        'amount': 1,
        'cid': 0,
        'condition': {
            'details': unsigned_subfulfillments_dict,
            'uri': threshold_sha256.condition_uri,
        },
        'owners_after': (alice.public_key, bob.public_key),
    }

    # The yet to be fulfilled fulfillment:
    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': None,
        'owners_before': (alice.public_key,),
    }

    # Craft the payload:
    handcrafted_car_creation_tx = {
        'asset': car_asset,
        'metadata': None,
        'operation': 'CREATE',
        'conditions': (condition,),
        'fulfillments': (fulfillment,),
        'version': 1,
    }

    # JSON: serialize the id-less transaction to a json formatted string
    # Generate the id, by hashing the encoded json formatted string representation of
    # the transaction:
    json_str_tx = json.dumps(
        handcrafted_car_creation_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    # SHA3: hash the serialized id-less transaction to generate the id
    car_creation_txid = sha3.sha3_256(json_str_tx.encode()).hexdigest()

    # add the id
    handcrafted_car_creation_tx['id'] = car_creation_txid

    # JSON: serialize the transaction-with-id to a json formatted string
    message = json.dumps(
        handcrafted_car_creation_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    # CRYPTO-CONDITIONS: sign the serialized transaction-with-id
    alice_ed25519.sign(message.encode(),
                       cryptoconditions.crypto.Ed25519SigningKey(alice.private_key))

    # CRYPTO-CONDITIONS: generate the fulfillment uri
    fulfillment_uri = alice_ed25519.serialize_uri()

    # add the fulfillment uri (signature)
    handcrafted_car_creation_tx['fulfillments'][0]['fulfillment'] = fulfillment_uri


Sending it over to a BigchainDB node:

.. code-block:: python

    from bigchaindb_driver import BigchainDB

    bdb = BigchainDB('http://bdb-server:9984/api/v1')
    returned_car_creation_tx = bdb.transactions.send(handcrafted_car_creation_tx)

Wait for some nano seconds, and check the status:

.. code-block:: python

    >>> bdb.transactions.status(returned_car_creation_tx['id'])
    {'status': 'valid'}

Handcrafting the ``'TRANSFER'`` transaction
-------------------------------------------

.. code-block:: python

    carol = generate_keypair()

    alice_ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=alice.public_key)

    bob_ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=bob.public_key)

    carol_ed25519 = cryptoconditions.Ed25519Fulfillment(public_key=carol.public_key)

    unsigned_fulfillments_dict = carol_ed25519.to_dict()

    condition_uri = carol_ed25519.condition.serialize_uri()

    condition = {
        'amount': 1,
        'cid': 0,
        'condition': {
            'details': unsigned_fulfillments_dict,
            'uri': condition_uri,
        },
        'owners_after': (carol.public_key,),
    }

    # The yet to be fulfilled fulfillments:
    fulfillment = {
        'fid': 0,
        'fulfillment': None,
        'input': {
            'txid': handcrafted_car_creation_tx['id'],
            'cid': 0,
        },
        'owners_before': (alice.public_key, bob.public_key),
    }

    # Craft the payload:
    handcrafted_car_transfer_tx = {
        'asset': {'id': car_asset['id']},
        'metadata': None,
        'operation': 'TRANSFER',
        'conditions': (condition,),
        'fulfillments': (fulfillment,),
        'version': 1,
    }

    # Generate the id, by hashing the encoded json formatted string
    # representation of the transaction:
    json_str_tx = json.dumps(
        handcrafted_car_transfer_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    car_transfer_txid = sha3.sha3_256(json_str_tx.encode()).hexdigest()

    handcrafted_car_transfer_tx['id'] = car_transfer_txid

    # Sign the transaction:
    message = json.dumps(
        handcrafted_car_transfer_tx,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=False,
    )

    alice_sk = cryptoconditions.crypto.Ed25519SigningKey(alice.private_key)

    bob_sk = cryptoconditions.crypto.Ed25519SigningKey(bob.private_key)

    threshold_sha256 = cryptoconditions.ThresholdSha256Fulfillment(threshold=2)

    threshold_sha256.add_subfulfillment(alice_ed25519)

    threshold_sha256.add_subfulfillment(bob_ed25519)

    alice_condition = threshold_sha256.get_subcondition_from_vk(alice.public_key)[0]

    bob_condition = threshold_sha256.get_subcondition_from_vk(bob.public_key)[0]

    alice_condition.sign(message.encode(), private_key=alice_sk)

    bob_condition.sign(message.encode(), private_key=bob_sk)

    fulfillment_uri = threshold_sha256.serialize_uri()

    handcrafted_car_transfer_tx['fulfillments'][0]['fulfillment'] = fulfillment_uri

Sending it over to a BigchainDB node:

.. code-block:: python

    bdb = BigchainDB('http://bdb-server:9984/api/v1')
    returned_car_transfer_tx = bdb.transactions.send(handcrafted_car_transfer_tx)

Wait for some nano seconds, and check the status:

.. code-block:: python

    >>> bdb.transactions.status(returned_car_transfer_tx['id'])
    {'status': 'valid'}



.. _sha3: https://github.com/tiran/pysha3
.. _cryptoconditions: https://github.com/bigchaindb/cryptoconditions
.. _cryptoconditions internet draft: https://tools.ietf.org/html/draft-thomas-crypto-conditions-01
.. _The Transaction Model: https://docs.bigchaindb.com/projects/server/en/latest/data-models/transaction-model.html
.. _The Transaction Schema: https://docs.bigchaindb.com/projects/server/en/latest/schema/transaction.html
