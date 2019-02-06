---
layout: default
title: ROS 2 Launch Static Descriptions - Implementation Considerations
permalink: articles/roslaunch.html
abstract:
  The launch system in ROS 2 aims to support extension of static descriptions, so as to easily allow both exposing new features of the underlying implementation, which may or may not be extensible itself, and introducing new markup languages. This document discusses several approaches for implementations to follow.
author: '[Michel Hidalgo](https://github.com/hidmic) [William Woodall](https://github.com/wjwwood)'
published: true
---

- This will become a table of contents (this text will be scraped).
{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Authors: {{ page.author }}

## Context

Static launch descriptions are an integral part to ROS 2 launch system, and the natural
path to transition from predominant ROS 1 `roslaunch` XML description. This document
describes parsing and integration approaches of different front ends i.e. different
markup languages, with a focus on extensibility and scalability.

## Proposed approaches

In the following, different approaches to the solution are described. This list
is by no means exhaustive.

It's worth to note some commonalities between them:

- All of them attempt to solve the problem in a general and extensible way to help
  the system scale with its community.

- All of them require a way to establish associations between entities, solved using
  unique reference id (usually, a human-readable name but that's not a requisite).

- All of them need some form of instantiation and/or parsing procedure registry for
  parsers to lookup. How that registry is populated and provided to the parser may vary.
  For instance, in Python class decorators may populate a global dict or even its import
  mechanism may be used if suitable, while in C++ convenience macros may expand into
  demangled registration hooks that can later be looked up by a dynamic linker (assuming
  launch entities libraries are shared libraries).

### Forward Description Mapping (FDM)

In FDM, the parser relies on a schema and well-known rules to map an static
description (markup) to implementation specific instances (objects).

#### Description Markup

Some description samples in different markup languages are provided below:

*XML*
```xml
<launch version="x.1.0">
  <action name="some-action" type="actions.ExecuteProcess">
    <arg name="cmd" value="/bin/ls"/>
  </action>
  <on_event type="events.ProcessExit" target="some-action">
    <action type="actions.LogInfo">
      <arg name="message" value="I'm done"/>
    </action>
  </on_event>
</launch>
```

*YAML*
```yml
launch:
  version: x.1.0
  actions:
    - type: actions.ExecuteProcess
      name: some-action
      args:
        cmd: /bin/ls
  handlers:
    - type: events.ProcessExit
      target: some-action
      actions:
        - type: actions.LogInfo
          args:
            message: "I'm done"        
```

### Advantages & Disadvantages

+ Straightforward to implement.

+ Launch implementations are completely unaware of the static description existence
  and its parsing process (to the extent that type agnostic instantiation mechanisms
  are available).

- Statically typed launch system implementations may require variant objects to deal
  with actions 

- Static descriptions are geared towards easing parsing, making them more uniform like
  a serialization format but also less user friendly.

- Care must be exercised to avoid coupling static descriptions with a given implementation.

### Forward Description Mapping plus Markup Sugars (FDM+)

A variation on FDM that allows launch entities to supply markup language specific hooks
to do their own parsing.

#### Description Markup

Some description samples in different markup languages are provided below:

*XML*
```xml
<launch version="x.1.0">
  <process name="some-action" cmd="/bin/ls">
     <on_exit>
        <log message="I'm done"/>
     </on_exit>
  </process>
</launch>
```

*YAML*
```yml
launch:
  version: x.1.0
  entities:
    - process:
        cmd: /bin/ls
        on_exit:
          - log: "I'm done"        
```

#### Parsing Hooks

Some code samples in different programming languages are provided below:

*Python*
```python
class SomeAction(LaunchDescriptionEntity):

   @launch.parse_from('xml')
   def parse(cls, element):
       ...
```

*C++*
```c++
class SomeAction : public LaunchDescriptionEntity {
public:
  static std::unique_ptr<SomeAction> parse_from_xml(const XmlElement& e) {
     ...
  }
};

LAUNCH_XML_PARSING_HOOK("some-action", SomeEntity::parse_from_xml);
```

#### Advantanges & Disadvantages

- Launch system implementations are aware of the parsing process, being completely
  involved with it if sugars are to be provided.

+ Allows leveraging the strenghts of each markup language.

- Opens the door to big differences in the representation of launch entities across
  different front end, and even within a given one by allowing the users to introduce
  multiple custom representations for the same concepts (e.g. a list of numbers).

- Care must be exercised to avoid coupling static descriptions with a given implementation.

+ Straightforward to implement.

### Abstract Description Parsing (ADP)

In ADP, the parser provides an abstract interface to the static description
and delegates parsing and instantiation to hooks registered by the implementation.

#### Description Markup

Some description samples in different markup languages are provided below:

*XML*
```xml
<launch version="x.1.0">
  <process name="some-action" cmd="/bin/ls">
    <on_exit>
      <log message="I'm done"/>
    </on_exit>
  </process>
</launch>
```

*YAML*
```yaml
launch:
  version: x.1.0
  entities:
    - process:
        cmd: /bin/ls
        on_exit:
          - log:
              message: 'I'm done'
```

#### Abstract Description

To be able to abstract away launch descriptions written in conceptually different
markup languages, the abstraction relies on the assumption that all launch system
implementations are built as object hierarchies. If that holds, then ultimately
all domain specific schemas and formats will just be different mappings of said
hierarchy.

In that context, a hierarchical, object-oriented representation of
the markup description can be built. Define an `Entity` object that has:

- a namespaced type;
- optionally a name, unique among the others;
- optionally a parent entity (i.e. unless it's the root entity);
- optionally one or more named attributes, whose values can either be
  entities or ordered sequences of them;

Some sample definitions in different programming languages are provided below:


*Python*
```python
class Entity:

   @property
   def type(self):
       pass

   @property
   def parent(self):
       pass

   @property
   def children(self):
       pass

   def __getattr__(self, name):
       pass
```

*C++*
```c++
namespace parsing {
class Entity {

  const std::string& type() const;

  const Entity& parent() const;

  const std::vector<Entity>& children() const;

  template<typename T>
  const T& get(const std::string& name) const { ... }

  const Entity& get(const std::string& name) const { ... }

  template<typename T>
  bool has(const std::string& name) const { ... }

  bool has(const std::string& name) const { ... }
};
}  // namespace parsing
```

It is up to each front end implementation to choose how to map these concepts
to the markup language. For instance, one could map both of the following
descriptions:

*XML*
```xml
<node name="my-node" package="demos" executable="talker">
  <param name="a" value="100."/>
  <param name="b" value="stuff"/>
</node>
```

*YAML*
```yaml
- node:
    name: my-node
    package: demos
    executable: talker
    param:
      - name: a
        value: 100.
      - name: a
        value: stuff
```

such that their associated parsing entity `e` exposes its data as follows:

*Python*
```python
e.type == 'node'
e.name == 'my-node'
hasattr(e, 'package') == true
e.param[0].name == 'a'
e.param[1].name == 'b'
```

*C++*
```c++
e.type() == "node"
e.get<std::string>("name") == "my-node"
e.has<std::string>("package") == true
auto params = e.get<std::vector<Entity>>("param")
params[0].get<std::string>("name") == "a"
params[1].get<std::string>("name") == "b"
```

Inherent ambiguities will arise from the mapping described above, e.g. nested
tags in an XML description may be understood either as children (as it'd be
the case for a grouping/scoping action) or attributes of the enclosing tag
associated entity (as it's the case in the example above). It is up to the
parsing procedures to disambiguate them.

#### Parsing Delegation

Each launch entity that is to be statically described must provide a
parsing procedure. 

*Python*
```python
@classmethod
def parse(
    cls,
    entity: parsing.Entity,
    parser: parsing.Parser
) -> LaunchDescriptionEntity:
    return cls(...)
```

*C++*
```c++
std::unique_ptr<LaunchDescriptionEntity>
parse(const parsing::Entity & e, const parsing::Parser & parser) {
   return std::make_unique<LaunchDescriptionEntity>(...);
}
```

As can be seen above, procedures inspect the description through the given
parsing entity, delegating further parsing of composed launch entities
to the parser. Delegation is thus recursive.

#### Procedure Provisioning

##### Manual Provisioning

In the simplest case, the user may explicitly provide their own parsing
procedure for each launch entity.

##### Automatic Derivation

If accurate type information is (somehow) available, reflection mechanisms
can aid derivation of a parsing procedure with no user intervention.

### Advantages & Disadvantages

The abstraction layer allows.

- Launch system implementations are aware of the parsing process.

+ The static description abstraction effectively decouples launch frontends
  and backends, allowing for completely independent development and full
  feature availability at zero cost.

- No markup language specific sugars are possible. REVISIT(hidmic): IMHO
  explicitly disallowing this is a good thing, it makes for more homogeneus
  descriptions and avoids proliferation of multiple representation of the
  same concepts (e.g. a list of strings).

+ The transfer function nature of the parsing procedure precludes the need
  for a rooted object type hierarchy in statically typed launch system
  implementations.

- Automatic parsing provisioning requires accurate type information, which
  may not be trivial to gather in some implementations.

- Trickier to implement.
