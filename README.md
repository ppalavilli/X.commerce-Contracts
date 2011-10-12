# X.commerce Contracts

Copyright (c) 2011 X.commerce

## License

By downloading any files from this repository, you are agreeing to the terms and conditions of the license that govern the X.commerce Contrarcts.  The license is available at [https://www.x.com/developers/xcommerce/products/developer-package/license](https://www.x.com/developers/xcommerce/products/developer-package/license).

## What's a Contract?

First, another term: ontology.  The X.commerce Ontology is the mechanism for specifying the common language, interaction standards and integration model for users of the X.commerce Fabric. The ontology is comprised of:

* Entities - e.g. Customer, Merchant, Order, Product
  * Attributes - e.g. Customer-ID, Product-SKU
  * Lifecycles - e.g. Order: Created, Approved, Rejected, etc.
  * Relationships - e.g. Order has 1..N Product
* Events - e.g. Order Approved Event
* Domain description and glossary - e.g. The words of the language

Contracts are the _machine readable_ realization of the ontology that governs the message-based interactions between capabilities via the X.commerce Fabric.  Contracts are defined using the [Avro](http://avro.apache.org/) Interface Description Language (IDL), in ".avdl" files.  You'll see that we also use Java-style annotations in the AVDL files to, for example, associate message topics on the Fabric to message schemas. Messages that travel across the Fabric are serialized using Avro, typically using its binary format.

## The Approach

Definition and evolution of the X.commerce Ontology, including the contracts, will be done in a community setting with X.commerce providing guiding principles and sponsorship.  The approach:

* The ontology will be maintained in this GitHub repository
* An officially published version of the ontology will be available in sandbox and production environments so capabilities compliant with this version can be deployed to those environments
* Anyone in the community (including partners, developers and eBay, Inc. employees) can fork this repository, work on changes and make a pull request to propose merging them back into the official version
* Initially X.commerce will have a core team of committers
* Over time, X.commerce will support domain-specific committers and expand the pool of committers to others in the community based on their contributions

The versioning scheme for contracts, including compatibility and deprecation rules, will be published soon.

## Current State

X.commerce is sharing our work on contracts for the ecosystem early - very early. The open approach described above is the obvious motivation.  The downside is that the current state of the contracts doesn't reflect some of the principles and practices that we believe are important to ensure robustness and maintainability within the ecosystem.  They do, however, reflect the currently running capabilities and applications that we demonstrated at Innovate 2011, so we're constantly learning.  You can help.

The first, and most noticeable, flaw in the contracts is the use of request/response terminology.  While the Fabric is asynchronous, the semantics of request/response are very familiar, so using this terminology was an expedient way to get an initial version of the contracts implemented.  Unfortunately, this approach leads us down a dangerous path _because_ the Fabric is asynchronous and message-based.  So, we'll be significantly refactoring the contracts we've published so far, and holding off others that we already have until they're in better shape.

Next, we plan to reorganize message topics are a "state-based" approach, wherein topic names reflect a declaration of a change in state of an entity, rather than sounding like queries, actions or operations.

Other flaws are more superficial, but should be addressed.  These include inconsistencies in naming, 

## What Now?

* Get the [X.commerce Developer Package](https://www.x.com/fabric-download)
* Try the X.commerce Innovate "Code to Capability" [tutorial](https://github.com/xcommerce/innovate-developer-demo)
* Fork the repo and make the ecosystem better!

## Details

### Compiling Contracts (AVDL -> AVPR)

The Eclipse plug-in in the [X.commerce Developer Package](https://www.x.com/fabric-download) lets you create capabilities directly from AVDL files.  However, if you're using a dynamic language with Avro - there are bindings for Python, Ruby, PHP - you'll want to compile an AVDL to an AVPR file that can be parsed by the AvroProtocol (or similarly-named) class in your language of choice.  To compile, you will need the the Avro tools jar built from the [Java Avro package](http://www.apache.org/dyn/closer.cgi/avro/).  Then you can compile like so:

    java -jar ~/packages/avro-src-1.5.1/lang/java/tools/target/avro-tools-1.5.1.jar idl <idl file>

### Language-specific Tweaks

We've made a couple of tweaks to both the PHP and Ruby Avro implementations to make them easier to use for our purposes.  Specifically, we use the format typically used for data file storage for encoding messages, which required us to add a way to "force" writing complete schemas into messages.  Both the PHP and Avro versions are in this repository, until we get appropriate patches accepted into the official open source releases.

Here's an example of this in action for both PHP and Ruby, referencing [Inventory.avdl](https://github.com/xcommerce/innovate-developer-demo/blob/master/Inventory.avdl) from the Innovate developer tutorial.

PHP:

    <?php

    include_once 'avro.php';

    $schema = file_get_contents($argv[1]);    // read an avpr; use Inventory.avdl

    $p = AvroProtocol::parse($schema);        // parse it

    $schemata = $p->schemata;                 // get all the schemas

    // combine the schemas we need in a union schema
    // the "true" parameter says "include all the schemas in the encoded message"

    $s = new AvroUnionSchema(array("Item", "Items"),
    			     "com.x.product.inventory", $schemata, true);

    $datum_writer = new AvroIODatumWriter($s);
    $strio = new AvroStringIO();
    $dw = new AvroDataIOWriter($strio, $datum_writer, $s);

    $i = array("sku" => "123", "title" => "title1", "currentPrice" => "price", "url" => "url", "dealOfTheDay" => "0");
    $is = array("items" => array($i));
    $dw->append($is);
    $dw->close();


Ruby:

    #!/usr/bin/env ruby

    require 'rubygems'
    require 'avro'

    protocol_text = File.open(ARGV[0], "rb").read    # read an avpr; use Inventory.avdl

    protocol = Avro::Protocol.parse(protocol_text)   # parse it

    schemas = {}
    protocol.types.map {|t| schemas[t.name] = t}     # get all the schemas

    # combine the schemas we need in a union schema
    # the "true" parameter says "include all the schemas in the encoded message"

    schema = Avro::Schema::UnionSchema.new(["Item", "Items"], "com.x.product.inventory", schemas, true)

    buffer = StringIO.new
    writer = Avro::IO::DatumWriter.new(schema)
    dw = Avro::DataFile::Writer.new(buffer, writer, schema)
    item = {"sku" => "123", "title" => "title", "currentPrice" => "price", "url" => "url", "dealOfTheDay" => "0"}
    items = {"items" => [item]}
    dw << items
    dw.close

