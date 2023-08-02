# Cedar Schema Syntax

## Related issues and PRs

- Reference Issues: 
- Implementation PR(s): 

## Timeline

- Start Date: 2023-07-24
- Date Entered FCP:
- Date Accepted:
- Date Landed:

## Summary

This document proposes a custom syntax for Cedar schemas. The syntax was developed with the following goals in mind:

* Use the same concepts in both the JSON-based and custom syntax, allowing them to be naturally inter-converted
* Reuse Cedar policy syntax as much as possible, to help with intuition
* When no Cedar policy syntax analogue exists, leverage ideas from other programming languages that favor readability
* Use as few syntactic concepts as possible; similar ideas in the schema should look similar syntactically

This is _not_ a proposal to _replace_ the JSON-based syntax; the aim is to provide an additional syntax that better satisfies the goals above. The JSON syntax is appropriate in many circumstances, such as when constructing schemas systematically. Adopting this proposal would put schemas on the same footing as Cedar policies, which have a custom and a JSON syntax.

## Basic example

Here is the proposed syntax for the [TinyTodo schema](https://github.com/cedar-policy/cedar-examples/blob/main/tinytodo/tinytodo.cedarschema.json), whose auto-formatted JSON version is 160 lines long.
```
entity Application;
entity User in [Team,Application] { name: String };
entity Team in [Team,Application];
entity List in [Application] {
    owner: User,
    name: String,
    readers: Team,
    editors: Team,
    tasks: Set<{name: String, id: Long, state: String}>
};

action CreateList, GetLists
    appliesTo { principal: [User], resource: [Application] };

action GetList, UpdateList, DeleteList, CreateTask, UpdateTask, DeleteTask, EditShares
    appliesTo { principal: [User], resource:[List] };
```
## Motivation

Cedar schemas for non-toy applications are hard to read and write. The root cause of the difficulty is the use of JSON:

* JSON has low information density. For three applications we’ve modeled — [TinyTodo](https://github.com/cedar-policy/cedar-examples/blob/main/tinytodo/tinytodo.cedarschema.json), [Document Cloud](https://github.com/cedar-policy/cedar-examples/blob/main/cedar-example-use-cases/document_cloud/schema.json), and [GitHub](https://github.com/cedar-policy/cedar-examples/blob/main/cedar-example-use-cases/github_example/schema.json) — schemas span 4-5 pages.
* JSON does not support comments. This means that any design intent for a schema cannot be expressed inline.

We believe that a custom syntax for Cedar schemas can help. It can be more information-dense, support comments, and leverage familiar syntactic constructs. We think customers will be happier with a custom syntax: we regularly hear favorable reviews of Cedar's custom policy syntax, as compared to a JSON-based syntax.

## Detailed Design

In what follows we first present the syntax by example. A full grammar is given at the end.

### Comments

Cedar schemas can use line-ending comments in the same style as Cedar, e.g., `// this is a comment`.

### Namespace format

```
namespace My::Namespace {
  ...
}
```

A Schema can have zero or more namespace declarations. The contents of each (`...` above) comprise entity, action, and common-type declarations, all of which are unordered; i.e., they are all allowed to mutually refer to one another (but see discussion on common types below). The recommended style is to put entity declarations first, followed by action declarations, with common type declarations interspersed as needed. Entity, action, and common types referenced in the same namespace in which they are defined need not be prefixed by that namespace.

Entity, action, and common-type declarations can appear on their own, outside of any namespace; in that case they are implicitly within the empty namespace.

### Entity declarations

Here is an example illustrating the entity declaration syntax.

```
entity User in [Group] {
    personalGroup: Group,
    delegate?: User,
    blocked: Set<User>,
};
entity Document {
    owner: User,
    isPrivate: Boolean,
    publicAccess: String,
    viewACL: DocumentShare,
    modifyACL: DocumentShare,
    manageACL: DocumentShare
};
entity DocumentShare;
entity Public in [DocumentShare];
```

The first declaration indicates that `User` is an entity type, can be a member of the `Group` type, and has two attributes: `personalGroup` with entity type `Group`, and `blocked` with type `Set<User>`.

To allow syntactic parity with common `type` declarations, discussed below, we can optionally add `=` just before the `{ ... }` defining an entity's attributes, e.g., `entity Dog = { name: String }` rather than `entity Dog { name: String }`. Allowing `=` also supports a future extension in which an entity's shape is a type other than a record.

Entity declarations may or may not have the `in ...` component, and they may or may not have attributes. A singleton list can be expressed without brackets, e.g., `entity User in Group = ...` Optional attributes are specified in the style of Typescript, with a `?` after their name. The last element of an the attribute list can optionally include a comma.

Entity types that share a definition can be declared together. For example, the following declares two entity types `User` and `Team`, both of which can be members of `Team` and both of which have a `name` attribute.
```
entity User, Team in [Team] { name: String };
```

### Action declarations

Here is an example illustrating the action declaration syntax.
```
action ReadActions;
action WriteActions;
action CreateDocument 
    appliesTo { principal: [User], resource: [Drive] };
action DeleteDocument in [WriteActions] 
    appliesTo { principal: [User], resource: [Document] };
action ViewDocument in [ReadActions] appliesTo {
    principal: [User,Public],
    resource: Document,
    context: {
        network: ipaddr,
        browser: String
    }
};
```

A declared action indicates either none, one, or both of the following:

1. What action groups it is a member of, using `in`
2. What principal, resource, and context types it may be coupled with in a request (if any), using `appliesTo`

Specifying `[]` for the `in` component is equivalent to omitting `in` entirely. An action that has no `appliesTo` component essentially names an action group: having no possible principal and/or resource types means there’s no way to submit a legal request with the action.

The `appliesTo` specificiation uses record syntax, where the "attributes" of the record may be any combination of `principal`, `resource`, and `context`, with the following constraints.
+ If `principal` and/or `resource` is given, the accompanying type must be either
  - a single entity type,
  - a list of entity types, or
  - the syntax `[*]`, which is interpreted as allowing _any_ entity type (equivalently, the "unspecified" entity).
+ If a `principal` or `resource` element is not given, that means that any entity can be used for that element in a request (and is equivalent, therefore, to it having type `[*]`).
+ If the type given with `principal` or `resource` is `[]`, that means that _no_ entity is permitted in a request with this action, which essentially means that the action can only be used as a group
+ If `context` is given, the accompanying type must be a record type. If it is not given, the type `{}` is assumed.
+ At least one of `principal`, `resource`, or `context` must be included if `appliesTo` is present; i.e., writing `appliesTo { }` is not allowed.

Here are two additional examples that follow these rules:

```
action ViewDocument in ReadActions appliesTo {
    resource: Document,
    context: {}
};
action Ping appliesTo {
    principal: [*],
    resource: [*]
};
```

Since actions are entities, action names are entity IDs, which can be arbitrary strings. Thus we admit the following more general syntax as well.

```
action "Delete Document $$" in ["Write Actions"] appliesTo {
    principal: [User],
    resource: [Document]
};
```

We anticipate future support for attributes as part of action entities. If/when they are supported, we can specify action attributes prior to the `appliesTo` part as an inline record using Cedar syntax. For example, here’s an extension of `ViewDocument` with action attributes.

```
action ViewDocument in [ReadActions] {
  version: 1,
  group: "meta"
} appliesTo {
  principal: [User,Public],
  resource: Document,
  context: {
    network: ipaddr,
    browser: String
  }
};
```

### Common types

You can define names for types (as in Typescript, OCaml, Haskell, and many other languages) and then use them in `action` and `entity` declarations. Here is an example:

```
type authcontext = {
    ip: ipaddr,
    is_authenticated: Boolean,
    timestamp: Long
};
entity Ticket {
  who: String,
  operation: Long,
  request: authcontext
};
action view appliesTo { context: authcontext };
action upload appliesTo { context: authcontext };
```
Note the use of `=` for `type` declarations, but the lack of (required) `=` for `entity` declarations; the `=` is needed because the definining type may not be a record, e.g., `type time = Long`.

As with `entity` and `action` declarations, the name implicitly includes the surrounding namespace as a prefix.

While common types can be declared anywhere within a namespace and safely referenced by entity and action declarations, they cannot refer to one another, to avoid introducing definitional cycles. (To allow mutual reference, we could make declaration order matter: `entity`, `action`, and other `type` declarations cannot refer to a common type before it is defined.)

### Mockup: Document Cloud

Here is a version of the [Document Cloud](https://github.com/cedar-policy/cedar-examples/blob/main/cedar-example-use-cases/document_cloud/schema.json) schema.

```
namespace DocCloud {
    entity User in [Group] {
        personalGroup: Group,
        blocked: Set<User>
    };
    entity Group in [DocumentShare] { 
        owner: User 
    };
    entity Document {
        owner: User,
        isPrivate: Boolean,
        publicAccess: String,
        viewACL: DocumentShare,
        modifyACL: DocumentShare,
        manageACL: DocumentShare
    };
    entity DocumentShare;
    entity Public in [DocumentShare];
    entity Drive;

    action CreateDocument, CreateGroup 
        appliesTo { principal: [User], resource: [Drive] };
    action ViewDocument 
        appliesTo { principal: [User,Public], resource: [Document] };
    action DeleteDocument, ModifyDocument, EditIsPrivate, AddToShareACL, EditPublicAccess 
        appliesTo { principal: [User], resource: [Document] };
    action ModifyGroup, DeleteGroup 
        appliesTo { principal: [User], resource: [Group] };
}
```

### Mockup: GitHub Model

Here is a version of the [GitHub](https://github.com/cedar-policy/cedar-examples/blob/main/cedar-example-use-cases/github_example/schema.json) schema.

```
namespace GitHub {
    entity User in [UserGroup,Team];
    entity UserGroup in [UserGroup];
    entity Repository {
        readers: UserGroup,
        triagers: UserGroup,
        writers: UserGroup,
        maintainers: UserGroup,
        admins: UserGroup   
    };
    entity Issue {
        repo: Repository,
        reporter: User
    };
    entity Org {
        members: UserGroup,
        owners: UserGroup,
        memberOfTypes: UserGroup
    };
    action pull, fork, push, add_reader, add_triager, add_writer, add_maintainer, add_admin 
        appliesTo { principal: [User], resource: [Repository] };
    action delete_issue, edit_issue, assign_issue 
        appliesTo { principal: [User], resource: [Issue] };
}
```

### Grammar

Here is a grammar for the proposed schema syntax.

The grammar applies the following the conventions. Capitalized words stand for grammar productions, and lexical tokens are given in all-caps. When productions or tokens match those in the Cedar policy grammar, we use the same names (e.g., `IDENT` and `Path`).

For grammar productions it uses `|` for alternatives, `[]` for optional productions, `()` for grouping, and `{}` for repetition of a form zero or more times.

Tokens are defined using regular expressions, where `[]` stands for character ranges; `|` stands for alternation; `*` , `+` , and `?` stand for zero or more, one or more, and zero or one occurrences, respectively; `~` stands for complement; and `-` stands for difference. The grammar ignores whitespace and comments.

```
Schema    := {Namespace}
Namespace := ('namespace' Path '{' {Decl} '}') | {Decl}
Decl      := Entity | Action | TypeDecl
Entity    := 'entity' Idents ['in' EntOrTyps] [['='] RecType] ';'
Action    := 'action' Names ['in' (Name | '[' [Names] ']')] [AppliesTo] ';'
TypeDecl  := 'type' IDENT '=' Type ';'
Type      := PRIMTYPE | IDENT | SetType | RecType
EntType   := Path
SetType   := 'Set' '<' Type '>'
RecType   := '{' [AttrDecls] '}'
AttrDecls := Name ['?'] ':' Type [',' | ',' AttrDecls]
AppliesTo := 'appliesTo' '{' AppDecls '}'
AppDecls  := VAR ':' EntOrTyps [',' | ',' AppDecls]
Path      := IDENT {'::' IDENT}
EntTypes  := Path {',' Path}
EntOrTyps := EntType | '[' [EntTypes] ']'
Name      := IDENT | STR
Names     := Name {',' Name}
Idents    := IDENT {',' IDENT}

IDENT     := ['_''a'-'z''A'-'Z']['_''a'-'z''A'-'Z''0'-'9']* - PRIMTYPE
STR       := Fully-escaped Unicode surrounded by '"'s
PRIMTYPE  := 'Long' | 'String' | 'Bool'
VAR       := 'principal' | 'action' | 'resource' | 'context'

WHITESPC  := Unicode whitespace
COMMENT   := '//' ~NEWLINE* NEWLINE
```

## Drawbacks

There are several reasons not to develop a custom syntax:

### Multiple formats can raise cognitive burden

Adding another schema format raises the bar for what customers need to know. They may ignore one format at first, but eventually they may need to learn both. For example, if one team uses the JSON format and another uses the custom format, but then the teams merge, they will each end up having to read/update the other's format. They may also fight about which format to move to.

Mitigating this problem is that it's easy and predictable to convert between the two formats, since the syntactic concepts line up very closely. The features you lose when converting from the new format to the JSON one would be 1/ any comments, and 2/ any use of intermingling of `action`, `entity`, and `type` declarations, since they must all be in their own sections in the JSON. Otherwise the features line up very closely and the only difference is syntax. We would expect to write conversion tools as part of implementing the new syntax (which are easily achieved by parsing in the new format and pretty-printing in JSON).

Another mitigating point is that it's early days for Cedar, and we can promote the custom syntax as the preferred choice. JSON should be used when writing tooling to auto-author or manipulate schemas, e.g., as part of a GUI. We have a [JSON syntax for Cedar policies](https://docs.cedarpolicy.com/json-format.html) for similar reasons, but it's the custom syntax that is front and center.

### Requirement of good tooling

As a practical matter, having a custom schema syntax will require that we develop high-quality tools to help authors write correct schemas. 

Parse error messages need to be of good quality, and report common issues such as missing curly braces, missing semi-colons, incorrect keywords, etc. A formatting tool can help ensure standardized presentation. An IDE plugin can put these things together to provide interactive feedback. For the JSON syntax, IDE plugins already exist to flag parse errors and to format documents uniformly. 

However, JSON plugins do not understand the semantics of Cedar schemas, so they cannot provide hints about semantic errors (e.g., that setting `principalTypes` to `[]` for an `action` effectively means the `action` cannot be used in a request). We could develop a linting tool that flags these issues, and it could be applied to both syntaxes. 

Note that the problem of matching up curly braces in JSON-formatted schemas is more acute than in the custom syntax, since it's common to have deep nesting levels where matching is hard to eyeball. For example, in the `TinyTodo` JSON schema, we have up to seven levels of direct nesting of curly braces, whereas the custom syntax has only three, and these levels are more visually isolated because of the other syntax in between them.

### Greater implementation cost

Supporting a new syntax is an extra implementation cost, including the new tools mentioned above, now and going forward. More code/tools means more potential for bugs. 

Mitigating this problem: The Rust internals already parse the JSON schema to a data structure, so adding a new format only involves adding a parser to this data structure and not, for example, making changes to the validator. We can use property-based testing to ensure that both formats are interconvertible, and correctly round-trip, i.e., that *parse(pretty-print(AST)) == AST*.

## Alternatives

One alternative would be to _replace_ the current JSON-based sytnax with the one in this proposal. This proposal would avoid the "cognitive burden" drawback mentioned above, but would be a disruptive, backward-incompatible change, and would lose the JSON format's benefits of existing tooling and easier programmatic schema construction.

Another alternative would be to adopt a [Yaml](https://en.wikipedia.org/wiki/YAML)-based syntax. This approach would meet our goals of greater information density and support for comments, and it would come with some existing tooling (such as IDE extensions). A downside of Yaml is that it provides _more_ than we need, with a lack of conciseness leading to confusing. We could make our own parser for a subset of Yaml we wish to support for schemas, but that may lead to a confusing user experience. Yaml's indentation-sensitive parsing also means that an indentation mistake will be silently accepted, leading to a confusing user experience. Our custom syntax is whitespace-insensitive, and having total control over the grammar means better context for error messages.

## Unresolved questions

One unresolved challenge is type name resolution. There are four kinds of type name:
1. Primitive types, e.g., `Long`, `String`, etc.
2. Extension types, currently just `ipaddr` and `decimal`
3. Entity types
4. Common types

The first two are built in. The last two are schema-declarable; they have an associated namespace prefix, but references within the defining namespace can drop the prefix.

In the current JSON syntax, common types may not overlap with primitive types; e.g., declaring a common type `Long` is disallowed. However, common types are permitted to overlap with extension types; e.g., declaring common type `ipaddr` is allowed. This works in the JSON because specifying an extension type requires an explicit `"type"` designator, as in the following:
```
"addr": {
    "ip": { "type": "Extension", "name": "ipaddr" },
}
```
If we _also_ had defined a common type `ipaddr` we would reference it by writing
```
"addr": {
    "myip": { "type": "ipaddr" },
}
```
Common types may also overlap with entity types, for similar reasons: To reference an entity type you must write `{ "type": "Entity", "name": "Foo" }` while to reference a common type you'd just write `{ "type": "Foo" }`.

The custom syntax does not require such designators, for readability. How should type names that are ambiguous therefore be resolved? For example, what should happen when a schema references type `ipaddr` when it also defines `entity ipaddr` and `type ipaddr` ? 

The main proposal already allows using namespace prefixes for disambiguation. If we had entity `ipaddr` and extension type `ipaddr` we could write `My::Namespace::ipaddr` to distinguish the former from the latter, assuming we are using a non-empty namespace. We can do likewise to distinguish entity type `Long` from primitive type `Long`. Namespaces are not sufficient to disambiguate entity types from common types, though, since they both can have namespace prefixes. Moreover, long namespace prefixes could create readability problems.

Here are two possible solutions:

### Typename prefixes

The first solution is to create type-name prefixes for the purpose of disambiguation, which parallel the `"type"` part of the JSON syntax. Proposal:
1. Extension types can be optionally prefixed with `+`, e.g., `+ipaddr` and `+decimal`
2. Entity types can be optionally prefixed with `&`, e.g., `&User` and `&My::Namespace::List`.

With these prefixes, we could write the following contrived schema:
```
entity ipaddr { name: String };
type ipaddr = Long;
entity container {
    issuer: &ipaddr, // entity ipaddr
    time: ipaddr,    // type ipaddr
    addr: +ipaddr    // extension type ipaddr
};
```
You would only write the prefix if there was a possibility for ambiguity. If you leave off a prefix but a name does not uniquely resolve, the schema parser will complain about the ambiguity.

### Forbidding type name overlap

As an alternative solution, we could forbid type-name overlaps entirely. Specifically, we could disallow creating any entity or common types that overlap with extension or primitive types, and we could disallow such types from mutually overlapping within the same namespace.

This solution would lead to better readability (due to fewer concepts imposed on the reader for interpreting type names), but would require imposing a similar restriction on the JSON form of the syntax, which is a breaking change.

As a middle ground, we could choose the first solution above, but _warn_ about overlapping type definitions, to discourage users from defining a common type with the same name as an entity or extension type.

