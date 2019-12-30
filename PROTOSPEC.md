# Protospec

## 8.4.1 EnqueueJob ( _queueName_, _job_, _arguments_ )

1.  Assert: Type(_queueName_) is String and its value is the name of a Job Queue
    recognized by this implementation.
2.  Assert: _job_ is the name of a Job.
3.  Assert: _arguments_ is a List that has the same number of elements as the
    number of parameters required by _job_.
4.  Let _callerContext_ be the running execution context.
5.  Let _callerRealm_ be _callerContext_'s Realm.
6.  Let _callerScriptOrModule_ be _callerContext_'s ScriptOrModule.
7.  <ins>Let _callerRec_ be EnvironmentRecord of _callerContext_'s
    LexicalEnvironment.</ins>
8.  <ins>Let _callerMeta_ be _callerRec_.\[\[CalleeMetadata]].</ins>
9.  Let _pending_ be PendingJob { \[\[Job]]: _job_, \[\[Arguments]]:
    _arguments_, \[\[Realm]]: _callerRealm_, \[\[ScriptOrModule]]:
    _callerScriptOrModule_, \[\[HostDefined]]: **undefined**<ins>,
    \[\[CallerMetadata]]: _callerMeta_</ins> }.
10. Perform any implementation or host environment defined processing of
    _pending_. This may include modifying the \[\[HostDefined]] field or any
    other field of pending.
11. Add _pending_ at the back of the Job Queue named by _queueName_.
12. Return NormalCompletion(`empty`).

## 8.6 RunJobs ( )

1. Perform ? InitializeHostDefinedRealm().
2. In an implementation-dependent manner, obtain the ECMAScript source texts
   (see clause 10) and any associated host-defined values for zero or more
   ECMAScript scripts and/or ECMAScript modules. For each such _sourceText_ and
   _hostDefined_, do
    1. If _sourceText_ is the source code of a script, then
        1. Perform EnqueueJob(**"ScriptJobs"**, ScriptEvaluationJob, «
           _sourceText_, _hostDefined_ »).
    2. Else,
        1. Assert: _sourceText_ is the source code of a module.
        2. Perform EnqueueJob(**"ScriptJobs"**, TopLevelModuleEvaluationJob, «
           _sourceText_, _hostDefined_ »).
3. Repeat,
    1. Suspend the running execution context and remove it from the execution
       context stack.
    2. Assert: The execution context stack is now empty.
    3. Let _nextQueue_ be a non-empty Job Queue chosen in an
       implementation-defined manner. If all Job Queues are empty, the result is
       implementation-defined.
    4. Let _nextPending_ be the PendingJob record at the front of _nextQueue_.
       Remove that record from nextQueue.
    5. Let _newContext_ be a new execution context.
    6. Set _newContext_'s Function to null.
    7. Set _newContext_'s Realm to _nextPending_.\[\[Realm]].
    8. Set _newContext_'s ScriptOrModule to _nextPending_.\[\[ScriptOrModule]].
    9. <ins>**NOTE: Absolutely unsure what to do here ¯\\\_(ツ)\_/¯** Set
       _newContext_'s LexicalEnvironment's
       EnvironmentRecord.\[\[CallerMetadata]] to !
       ObjectCreate(_nextPending_.\[\[CallerMetadata]]).</ins>
    10. Push _newContext_ onto the execution context stack; _newContext_ is now
        the running execution context.
    11. Perform any implementation or host environment defined job
        initialization using _nextPending_.
    12. Let _result_ be the result of performing the abstract operation named by
        _nextPending_.\[\[Job]] using the elements of
        _nextPending_.\[\[Arguments]] as its arguments.
    13. If _result_ is an abrupt completion, perform HostReportErrors(«
        _result_.\[\[Value]] »).

## 9.2.1.1 PrepareForOrdinaryCall ( _F_, _newTarget_ )

1.  Assert: Type(_newTarget_) is Undefined or Object.
2.  Let _callerContext_ be the running execution context.
3.  Let _calleeContext_ be a new ECMAScript code execution context.
4.  Set the Function of _calleeContext_ to _F_.
5.  Let _calleeRealm_ be _F_.\[\[Realm]].
6.  Set the Realm of _calleeContext_ to _calleeRealm_.
7.  Set the ScriptOrModule of _calleeContext_ to _F_.\[\[ScriptOrModule]].
8.  Let _localEnv_ be NewFunctionEnvironment(_F_, _newTarget_).
9.  <ins>Perform PrepareCelleeEnvironment(_callerContext_,
    _calleeContext_).</ins>
10. Set the LexicalEnvironment of _calleeContext_ to _localEnv_.
11. Set the VariableEnvironment of _calleeContext_ to _localEnv_.
12. If _callerContext_ is not already suspended, suspend _callerContext_.
13. Push _calleeContext_ onto the execution context stack; _calleeContext_ is
    now the running execution context.
14. NOTE: Any exception objects produced after this point are associated with
    _calleeRealm_.
15. Return _calleeContext_.

## <ins>?? PrepareCelleeEnvironment ( _callerContext_, _calleeContext_ )</ins>

When the PrepareCalleeEnvironment abstract operation is called with execution
context _callerContext_ and execution context _calleeContext_, the following
steps are taken:

1. Let _callerRec_ be EnvironmentRecord of _callerContext_'s LexicalEnvironment.
2. Let _calleeRec_ be EnvironmentRecord of _calleeContext_'s LexicalEnvironment.
3. Set _calleeRec_.\[\[CallerMetadata]] to !
   ObjectCreate(_callerRec_.\[\[CalleeMetadata]]).
4. Return NormalCompletion(`empty`).

## 9.3.1 \[\[Call]] ( _thisArgument_, _argumentsList_ )

## ?? EnvironmentMetadata

**TODO:** _New other property of Global Object._

## <ins>?? SetEnvironmentMetadata ( _propertyKey_, _value_ )</ins>

1. Let _key_ be ? ToPropertyKey(_propertyKey_).
2. Let _envRec_ be EnvironmentRecord of current execution context's
   LexicalEnvironment.
3. Let _callerMeta_ be _envRec_.\[\[CallerMetadata]].
4. Let _calleeMeta_ be _envRec_.\[\[CalleeMetadata]].
5. Perform ! _callerMeta_.\[\[Set]](_key_, _value_, _callerMeta_).
6. Perform ! _calleeMeta_.\[\[Set]](_key_, _value_, _calleeMeta_).
7. Return NormalCompletion(`empty`).

> **NOTE:** A property must be set in both caller and callee metadata. Caller
> metadata takes priority when resding properties using HasEnvironmentMetadata
> and GetEnvironmentMetadata, and callee metadata used to inherit from when
> passing to functions called from current context.

## <ins>?? HasEnvironmentMetadata ( _propertyKey_ )</ins>

1. Let _key_ be ? ToPropertyKey(_propertyKey_).
2. Let _envRec_ be EnvironmentRecord of current execution context's
   LexicalEnvironment.
3. Let _callerMeta_ be _envRec_.\[\[CallerMetadata]].
4. Let _calleeMeta_ be _envRec_.\[\[CalleeMetadata]].
5. If ! _callerMeta_.\[\[HasProperty]](_key_) is **true**, then
    1. Return **true**.
6. Else,
    1. Return ! _calleeMeta_.\[\[HasProperty]](_key_).

## <ins>?? GetEnvironmentMetadata ( _propertyKey_ )</ins>

1. Let _key_ be ? ToPropertyKey(_propertyKey_).
2. Let _envRec_ be EnvironmentRecord of current execution context's
   LexicalEnvironment.
3. Let _callerMeta_ be _envRec_.\[\[CallerMetadata]].
4. Let _calleeMeta_ be _envRec_.\[\[CalleeMetadata]].
5. If ! _callerMeta_.\[\[HasProperty]](_key_) is **true**, then
    1. Return ! _callerMeta_.\[\[Get]](_key_, _callerMeta_).
6. Else,
    1. Return ! _calleeMeta_.\[\[Get]](_key_, _calleeMeta_).

## ?? DeleteEnvironmentMetadata ( _propertyKey_ )
