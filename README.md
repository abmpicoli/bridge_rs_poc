Bridge proof-of-concept project.
================================

This project is the founding stone of the bridge project I have at work.

The original project allows a developer to implement a java application that
performs data collection, transformation and transfer without the need
to write java code.

Instead, the developer will simply create a set of property files and xsl 
transforms to process the data.

Data collection may happen in two ways:

1. **active by schedule** : every X minutes a data collection based on an existing state
  is executed.

    (For example, every minute, a state.properties file is collected showing the last
     record retrived. Then a sequence of data collection and transfers is performed.

     After a record is retrieved, the state is updated so if an application restart is 
     done, the solution can return from where it was stopped.

1. **as a web-service listener**: the application receives an inbound message, and Then
   perform the operations to deliver into another application.


And the actual operation is performed by a chain of **providers**:

Example:
* a HttpProvider, will perform an http query against an endpoint to either send or receive data.
* a SQLProvider will perform queries against a database (be selects, inserts or updates);
* A File provider will write content into a file in the filesystem.
* A StateProvider reads and save to state files, to keep state.

Data collection is performed by **consumers**: 

* A HttpConsumer is a web-service listener.
* A ScheduleConsumer simply starts a provider chain at specified intervals. 

Data is canonicalized into a w3c DOM document structure, that is the "internal"
structure of the data. This is done by *Parsers*, that receives a binary input stream,
and a charset specification, and turn it into a DOM document.

The input data is *always* converted into a DOM document, even the input is a json.

This is done by using xslt and the json-to-xml() function. [This paper](https://www.saxonica.com/papers/xmlprague-2016mhk.pdf) explains it in details.

The reason why this is done is because the glue between the processor chains are xslt transforms:

* Each processor step decides what happens next, by specifying an xpath that say what the next step is.

* Each processor in the chain has a unique name that is used in the configuration file.

* The configuration file is a java properties file that specify initialization properties, 
  instantiation options and *what* is instantiated. This is done in java by reflection.

  > which is not feasible in rust, being a compiled language. So, thinking about the solution itself,
  > I'm wondering if this could be changed by as single main.rs file with very simple orchestration
  > commands say `let stepx = http_processor::new().with_url("http://www.google.com")` 
  
* The orchestration between processor steps is done by the xsl transforms. An action xslt has the 
  role of:

  * collect data from the inbound message: it will become nodes in the dom tree.
  * define what the next step is, by specifying the next processor in the chain by querying for 
    a specific xpath in the DOM tree after the transform.

Finally, there is a response xslt that defines the final transform to return to a user, or to use in the next
step.

The problems with the current java framework:
-------------------------------------

* XSL is turing complete. But it is a pain in the a**. The developers tend to suffer trying to make even the simplest  operations, even tending to make recursive functions that bloats memory consumption to new steps.

* Sensitive data disclosure: there may be sensitive data in the inbound content, that may be disclosed in the output of xslt transforms. And the output of the xslt transforms is needed to diagnose issues.

* It is difficult to test processors in isolation.

* The full message content is kept in memory as potentially multiple DOM documents, which 
  may cause high memory consumption.

* XSL transforms may make recursive calls which bloats memory consumption even more.

* Processing is mandatorily sequential: a single processor step needs to be processed fully and turned into action
  before going to the next step. 

* Installation is somewhat complex:

  * The solution must run inside a tomcat: so a java environment with tomcat must be installed;
  * A full blown jar manifest 
