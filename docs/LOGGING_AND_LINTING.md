# FamilyTales Logging and Code Quality Guide

This guide covers logging strategies and code quality tools for maintaining high standards across the FamilyTales codebase.

## Table of Contents
- [Rust Logging with Tracing](#rust-logging-with-tracing)
- [Flutter Logging Strategy](#flutter-logging-strategy)
- [Rust Linting with Clippy](#rust-linting-with-clippy)
- [Flutter/Dart Linting](#flutterdart-linting)
- [Error Tracking with Sentry](#error-tracking-with-sentry)
- [Log Aggregation Strategy](#log-aggregation-strategy)

## Rust Logging with Tracing

### Setup and Configuration

```toml
# Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json", "time"] }
tracing-appender = "0.2"
tracing-opentelemetry = "0.22"
opentelemetry = "0.21"
opentelemetry-otlp = "0.14"
```

### Initializing Tracing

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing() -> Result<()> {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));

    let formatting_layer = tracing_subscriber::fmt::layer()
        .with_target(false)
        .with_thread_ids(true)
        .with_thread_names(true);

    // Different formatters for different environments
    let subscriber = match std::env::var("ENV").as_deref() {
        Ok("production") => {
            let json_layer = formatting_layer
                .json()
                .with_current_span(true)
                .with_span_list(true);
            
            tracing_subscriber::registry()
                .with(env_filter)
                .with(json_layer)
        }
        _ => {
            let pretty_layer = formatting_layer.pretty();
            
            tracing_subscriber::registry()
                .with(env_filter)
                .with(pretty_layer)
        }
    };

    subscriber.init();
    Ok(())
}
```

### Structured Logging Patterns

```rust
use tracing::{debug, error, info, instrument, warn};

#[instrument(skip(db_pool), fields(family_id = %family_id))]
pub async fn process_story(
    story_id: Uuid,
    family_id: Uuid,
    db_pool: &PgPool,
) -> Result<ProcessedStory> {
    info!("Starting story processing");
    
    // Log with structured fields
    info!(
        story_id = %story_id,
        family_id = %family_id,
        "Processing story for family"
    );
    
    // Log timing information
    let start = Instant::now();
    let result = ocr_service.process_image(&image_path).await?;
    
    info!(
        duration_ms = start.elapsed().as_millis(),
        confidence = result.confidence,
        "OCR processing completed"
    );
    
    // Error logging with context
    if let Err(e) = audio_service.generate_audio(&text).await {
        error!(
            error = %e,
            story_id = %story_id,
            "Failed to generate audio"
        );
        return Err(e);
    }
    
    Ok(processed_story)
}
```

### Log Levels and When to Use Them

```rust
// ERROR - Something failed that shouldn't have
error!(
    user_id = %user_id,
    error = %e,
    "Failed to authenticate user"
);

// WARN - Something unexpected but recoverable
warn!(
    retry_count = retry_count,
    "Mux API rate limit reached, retrying"
);

// INFO - Important business events
info!(
    story_id = %story_id,
    family_id = %family_id,
    "New story created"
);

// DEBUG - Detailed information for debugging
debug!(
    query = %query,
    params = ?params,
    "Executing database query"
);

// TRACE - Very detailed, often noisy information
trace!(
    headers = ?headers,
    "HTTP request headers"
);
```

### Context and Spans

```rust
use tracing::{span, Level};

pub async fn handle_family_request(req: Request) -> Response {
    let span = span!(
        Level::INFO,
        "handle_family_request",
        method = %req.method(),
        path = %req.uri().path(),
        request_id = %Uuid::new_v4(),
    );
    
    async move {
        // All logs within this block include span context
        info!("Processing request");
        
        let family_id = extract_family_id(&req)?;
        span.record("family_id", &family_id.to_string());
        
        // Nested spans for sub-operations
        let stories = {
            let span = span!(Level::DEBUG, "fetch_stories");
            async move {
                story_service.get_family_stories(family_id).await
            }.instrument(span).await?
        };
        
        info!(story_count = stories.len(), "Request completed");
        Ok(Response::new(stories))
    }
    .instrument(span)
    .await
}
```

## Flutter Logging Strategy

### Setup

```yaml
# pubspec.yaml
dependencies:
  logger: ^2.0.0
  logging: ^1.2.0
  sentry_flutter: ^7.14.0
```

### Logger Configuration

```dart
import 'package:logger/logger.dart';

class AppLogger {
  static late Logger _logger;
  
  static void init({bool isProduction = false}) {
    _logger = Logger(
      printer: isProduction 
          ? JsonPrinter() 
          : PrettyPrinter(
              methodCount: 2,
              errorMethodCount: 8,
              lineLength: 120,
              colors: true,
              printEmojis: true,
              printTime: true,
            ),
      level: isProduction ? Level.info : Level.debug,
      filter: ProductionFilter(),
      output: MultiOutput([
        ConsoleOutput(),
        if (isProduction) SentryOutput(),
      ]),
    );
  }
  
  static Logger get logger => _logger;
}

// Custom output for Sentry
class SentryOutput extends LogOutput {
  @override
  void output(OutputEvent event) {
    if (event.level.index >= Level.warning.index) {
      final message = event.lines.join('\n');
      Sentry.captureMessage(
        message,
        level: _mapLevel(event.level),
      );
    }
  }
  
  SentryLevel _mapLevel(Level level) {
    switch (level) {
      case Level.error:
        return SentryLevel.error;
      case Level.warning:
        return SentryLevel.warning;
      default:
        return SentryLevel.info;
    }
  }
}
```

### Logging Patterns in Flutter

```dart
import 'package:familytales/core/logging.dart';

class StoryService {
  final _logger = AppLogger.logger;
  
  Future<Story> createStory({
    required String title,
    required String content,
    required String familyId,
  }) async {
    _logger.i('Creating story', {
      'title': title,
      'familyId': familyId,
      'contentLength': content.length,
    });
    
    try {
      final stopwatch = Stopwatch()..start();
      
      final story = await _apiClient.createStory(
        title: title,
        content: content,
        familyId: familyId,
      );
      
      _logger.i('Story created successfully', {
        'storyId': story.id,
        'duration': stopwatch.elapsedMilliseconds,
      });
      
      return story;
    } catch (e, stackTrace) {
      _logger.e(
        'Failed to create story',
        error: e,
        stackTrace: stackTrace,
        {
          'familyId': familyId,
          'title': title,
        },
      );
      rethrow;
    }
  }
}
```

### Widget Lifecycle Logging

```dart
class StoryDetailPage extends ConsumerStatefulWidget {
  @override
  _StoryDetailPageState createState() => _StoryDetailPageState();
}

class _StoryDetailPageState extends ConsumerState<StoryDetailPage> {
  final _logger = AppLogger.logger;
  
  @override
  void initState() {
    super.initState();
    _logger.d('StoryDetailPage initialized');
  }
  
  @override
  void dispose() {
    _logger.d('StoryDetailPage disposed');
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return ErrorBoundary(
      onError: (error, stackTrace) {
        _logger.e(
          'Widget build error',
          error: error,
          stackTrace: stackTrace,
        );
      },
      child: _buildContent(),
    );
  }
}
```

## Rust Linting with Clippy

### Configuration

```toml
# Cargo.toml
[workspace.lints.clippy]
# Deny
future_not_send = "deny"
large_futures = "deny"
missing_errors_doc = "deny"
missing_panics_doc = "deny"
panic = "deny"
unreachable_pub = "deny"
unused_async = "deny"

# Warn
pedantic = "warn"
nursery = "warn"
cargo = "warn"
cognitive_complexity = "warn"
missing_const_for_fn = "warn"

# Allow
module_name_repetitions = "allow"
must_use_candidate = "allow"
```

### Custom Clippy Rules

```rust
// clippy.toml
cognitive-complexity-threshold = 20
too-many-arguments-threshold = 7
too-many-lines-threshold = 100
type-complexity-threshold = 250
```

### Common Clippy Patterns

```rust
// Use clippy suggestions for better code
#![warn(clippy::all, clippy::pedantic)]
#![allow(clippy::module_name_repetitions)]

// Example: Prefer explicit error handling
// Bad
fn process_data(data: &str) -> String {
    data.parse().unwrap()
}

// Good
fn process_data(data: &str) -> Result<String, ParseError> {
    data.parse()
}

// Example: Use more efficient methods
// Bad
let exists = vec.iter().find(|x| x == &target).is_some();

// Good
let exists = vec.contains(&target);

// Example: Avoid unnecessary allocations
// Bad
fn get_message(name: &str) -> String {
    format!("Hello, {}!", name).to_string()
}

// Good
fn get_message(name: &str) -> String {
    format!("Hello, {name}!")
}
```

### Running Clippy

```bash
# Run clippy with all features
cargo clippy --all-features -- -D warnings

# Run clippy with automatic fixes
cargo clippy --fix

# Run clippy on specific workspace member
cargo clippy -p familytales-api -- -D warnings
```

## Flutter/Dart Linting

### Configuration

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  strong-mode:
    implicit-casts: false
    implicit-dynamic: false
  
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
    - "lib/generated/**"
  
  errors:
    missing_required_param: error
    missing_return: error
    todo: warning
    deprecated_member_use_from_same_package: ignore

linter:
  rules:
    # Error rules
    avoid_dynamic_calls: true
    avoid_empty_else: true
    avoid_print: true
    avoid_relative_lib_imports: true
    avoid_returning_null_for_future: true
    avoid_slow_async_io: true
    avoid_type_to_string: true
    avoid_types_as_parameter_names: true
    avoid_web_libraries_in_flutter: true
    cancel_subscriptions: true
    close_sinks: true
    
    # Style rules
    always_declare_return_types: true
    always_put_control_body_on_new_line: true
    always_require_non_null_named_parameters: true
    annotate_overrides: true
    avoid_bool_literals_in_conditional_expressions: true
    avoid_catches_without_on_clauses: true
    avoid_catching_errors: true
    avoid_classes_with_only_static_members: true
    avoid_double_and_int_checks: true
    avoid_field_initializers_in_const_classes: true
    avoid_implementing_value_types: true
    avoid_js_rounded_ints: true
    avoid_null_checks_in_equality_operators: true
    avoid_positional_boolean_parameters: true
    avoid_private_typedef_functions: true
    avoid_redundant_argument_values: true
    avoid_renaming_method_parameters: true
    avoid_return_types_on_setters: true
    avoid_returning_null: true
    avoid_returning_null_for_void: true
    avoid_returning_this: true
    avoid_setters_without_getters: true
    avoid_shadowing_type_parameters: true
    avoid_single_cascade_in_expression_statements: true
    avoid_unnecessary_containers: true
    avoid_unused_constructor_parameters: true
    avoid_void_async: true
    await_only_futures: true
    camel_case_extensions: true
    camel_case_types: true
    cascade_invocations: true
    constant_identifier_names: true
    curly_braces_in_flow_control_structures: true
    directives_ordering: true
    do_not_use_environment: true
    empty_catches: true
    empty_constructor_bodies: true
    exhaustive_cases: true
    file_names: true
    flutter_style_todos: true
    hash_and_equals: true
    implementation_imports: true
    join_return_with_assignment: true
    leading_newlines_in_multiline_strings: true
    library_names: true
    library_prefixes: true
    missing_whitespace_between_adjacent_strings: true
    no_adjacent_strings_in_list: true
    no_duplicate_case_values: true
    no_logic_in_create_state: true
    no_runtimeType_toString: true
    non_constant_identifier_names: true
    null_closures: true
    only_throw_errors: true
    overridden_fields: true
    package_names: true
    package_prefixed_library_names: true
    parameter_assignments: true
    prefer_adjacent_string_concatenation: true
    prefer_asserts_in_initializer_lists: true
    prefer_asserts_with_message: true
    prefer_collection_literals: true
    prefer_conditional_assignment: true
    prefer_const_constructors: true
    prefer_const_constructors_in_immutables: true
    prefer_const_declarations: true
    prefer_const_literals_to_create_immutables: true
    prefer_constructors_over_static_methods: true
    prefer_contains: true
    prefer_equal_for_default_values: true
    prefer_final_fields: true
    prefer_final_in_for_each: true
    prefer_final_locals: true
    prefer_for_elements_to_map_fromIterable: true
    prefer_function_declarations_over_variables: true
    prefer_generic_function_type_aliases: true
    prefer_if_elements_to_conditional_expressions: true
    prefer_if_null_operators: true
    prefer_initializing_formals: true
    prefer_inlined_adds: true
    prefer_int_literals: true
    prefer_interpolation_to_compose_strings: true
    prefer_is_empty: true
    prefer_is_not_empty: true
    prefer_is_not_operator: true
    prefer_iterable_whereType: true
    prefer_mixin: true
    prefer_null_aware_operators: true
    prefer_single_quotes: true
    prefer_spread_collections: true
    prefer_typing_uninitialized_variables: true
    prefer_void_to_null: true
    provide_deprecation_message: true
    recursive_getters: true
    sized_box_for_whitespace: true
    slash_for_doc_comments: true
    sort_child_properties_last: true
    sort_constructors_first: true
    sort_pub_dependencies: true
    sort_unnamed_constructors_first: true
    test_types_in_equals: true
    throw_in_finally: true
    type_annotate_public_apis: true
    type_init_formals: true
    unawaited_futures: true
    unnecessary_await_in_return: true
    unnecessary_brace_in_string_interps: true
    unnecessary_const: true
    unnecessary_getters_setters: true
    unnecessary_lambdas: true
    unnecessary_new: true
    unnecessary_null_aware_assignments: true
    unnecessary_null_checks: true
    unnecessary_null_in_if_null_operators: true
    unnecessary_nullable_for_final_variable_declarations: true
    unnecessary_overrides: true
    unnecessary_parenthesis: true
    unnecessary_raw_strings: true
    unnecessary_statements: true
    unnecessary_string_escapes: true
    unnecessary_string_interpolations: true
    unnecessary_this: true
    unrelated_type_equality_checks: true
    unsafe_html: true
    use_build_context_synchronously: true
    use_full_hex_values_for_flutter_colors: true
    use_function_type_syntax_for_parameters: true
    use_is_even_rather_than_modulo: true
    use_key_in_widget_constructors: true
    use_late_for_private_fields_and_variables: true
    use_named_constants: true
    use_raw_strings: true
    use_rethrow_when_possible: true
    use_setters_to_change_properties: true
    use_string_buffers: true
    use_to_and_as_if_applicable: true
    valid_regexps: true
    void_checks: true
```

### Custom Lint Rules

```dart
// lib/analysis_options.dart
import 'package:custom_lint_builder/custom_lint_builder.dart';

PluginBase createPlugin() => _FamilyTalesLinter();

class _FamilyTalesLinter extends PluginBase {
  @override
  List<LintRule> getLintRules(CustomLintConfigs configs) => [
        NoDirectApiCalls(),
        RequireErrorHandling(),
        UseStructuredLogging(),
      ];
}

// Example custom rule
class NoDirectApiCalls extends DartLintRule {
  NoDirectApiCalls() : super(code: _code);

  static const _code = LintCode(
    name: 'no_direct_api_calls',
    problemMessage: 'Use service classes instead of direct API calls',
  );

  @override
  void run(
    CustomLintResolver resolver,
    ErrorReporter reporter,
    CustomLintContext context,
  ) {
    context.registry.addMethodInvocation((node) {
      if (node.methodName.name == 'http.get' ||
          node.methodName.name == 'http.post') {
        reporter.reportErrorForNode(_code, node);
      }
    });
  }
}
```

## Error Tracking with Sentry

### Rust Integration

```rust
use sentry::{ClientOptions, IntoDsn};

pub fn init_sentry() -> sentry::ClientInitGuard {
    let dsn = std::env::var("SENTRY_DSN")
        .expect("SENTRY_DSN must be set");
    
    let environment = std::env::var("ENV")
        .unwrap_or_else(|_| "development".to_string());
    
    sentry::init((
        dsn,
        ClientOptions {
            release: sentry::release_name!(),
            environment: Some(environment.into()),
            sample_rate: 1.0,
            traces_sample_rate: 0.1,
            attach_stacktrace: true,
            send_default_pii: false,
            before_send: Some(Arc::new(|event| {
                // Filter out sensitive data
                Some(filter_sensitive_data(event))
            })),
            ..Default::default()
        },
    ))
}

// Middleware for automatic error capture
pub fn sentry_middleware() -> SentryHttpLayer {
    SentryHttpLayer::new_from_top()
        .capture_server_errors(true)
        .capture_client_errors(false)
}

// Manual error capture with context
pub fn report_error(error: &AppError, context: &HashMap<String, String>) {
    sentry::with_scope(
        |scope| {
            scope.set_tag("module", "story_processing");
            for (key, value) in context {
                scope.set_extra(key, value.clone().into());
            }
        },
        || {
            sentry::capture_error(error);
        },
    );
}
```

### Flutter Integration

```dart
import 'package:sentry_flutter/sentry_flutter.dart';

Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = const String.fromEnvironment('SENTRY_DSN');
      options.environment = const String.fromEnvironment(
        'ENV',
        defaultValue: 'development',
      );
      options.tracesSampleRate = 0.1;
      options.attachStacktrace = true;
      options.sendDefaultPii = false;
      
      // Configure integrations
      options.integrations.add(LoggingIntegration());
      options.integrations.add(FlutterErrorIntegration());
      
      // Before send callback
      options.beforeSend = (event, {hint}) async {
        // Filter sensitive data
        return filterSensitiveData(event);
      };
      
      // Performance monitoring
      options.tracesSampler = (samplingContext) {
        final name = samplingContext.transactionContext.name;
        if (name == 'StoryUpload') {
          return 1.0; // Always trace story uploads
        }
        return 0.1; // 10% for everything else
      };
    },
    appRunner: () => runApp(MyApp()),
  );
}

// Capture errors with context
void reportError(dynamic error, StackTrace? stackTrace, {
  Map<String, dynamic>? extra,
  String? userMessage,
}) {
  Sentry.captureException(
    error,
    stackTrace: stackTrace,
    withScope: (scope) {
      scope.setTag('ui.component', 'story_viewer');
      if (extra != null) {
        extra.forEach((key, value) {
          scope.setExtra(key, value);
        });
      }
      if (userMessage != null) {
        scope.setContext('user_feedback', {'message': userMessage});
      }
    },
  );
}

// Performance monitoring
Future<T> trackPerformance<T>(
  String operation,
  Future<T> Function() task,
) async {
  final transaction = Sentry.startTransaction(
    operation,
    'task',
  );
  
  try {
    final result = await task();
    transaction.finish(status: const SpanStatus.ok());
    return result;
  } catch (error) {
    transaction.finish(status: const SpanStatus.internalError());
    rethrow;
  }
}
```

## Log Aggregation Strategy

### Architecture

```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash

volumes:
  es_data:
```

### Logstash Configuration

```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
  
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  # Parse Rust logs
  if [service] == "familytales-api" {
    json {
      source => "message"
      target => "rust_log"
    }
    
    mutate {
      add_field => {
        "[@metadata][target_index]" => "familytales-api-%{+YYYY.MM.dd}"
      }
    }
  }
  
  # Parse Flutter logs
  if [service] == "familytales-app" {
    json {
      source => "message"
      target => "flutter_log"
    }
    
    mutate {
      add_field => {
        "[@metadata][target_index]" => "familytales-app-%{+YYYY.MM.dd}"
      }
    }
  }
  
  # Extract common fields
  mutate {
    add_field => {
      "timestamp" => "%{[@timestamp]}"
      "environment" => "%{[fields][environment]}"
    }
  }
  
  # Drop sensitive fields
  mutate {
    remove_field => ["password", "token", "api_key"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][target_index]}"
  }
  
  # Alert on errors
  if [level] == "ERROR" {
    email {
      to => "alerts@familytales.com"
      subject => "Error in %{service}"
      body => "Error: %{message}\nStack: %{stack_trace}"
    }
  }
}
```

### Filebeat Configuration

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
  - type: docker
    containers:
      ids:
        - "*"
    processors:
      - add_docker_metadata: ~
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

### Log Retention and Rotation

```bash
# Elasticsearch index lifecycle management
PUT _ilm/policy/familytales_logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50GB"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Monitoring Dashboards

```json
// Kibana dashboard configuration
{
  "version": "8.11.0",
  "objects": [
    {
      "id": "familytales-overview",
      "type": "dashboard",
      "attributes": {
        "title": "FamilyTales Overview",
        "panels": [
          {
            "gridData": {"x": 0, "y": 0, "w": 24, "h": 15},
            "type": "visualization",
            "id": "error-rate-timeline"
          },
          {
            "gridData": {"x": 24, "y": 0, "w": 24, "h": 15},
            "type": "visualization",
            "id": "request-latency-histogram"
          },
          {
            "gridData": {"x": 0, "y": 15, "w": 48, "h": 20},
            "type": "search",
            "id": "recent-errors"
          }
        ]
      }
    }
  ]
}
```

### Best Practices

1. **Log Levels**
   - Use appropriate log levels consistently
   - ERROR: System failures requiring attention
   - WARN: Unexpected but handled situations
   - INFO: Important business events
   - DEBUG: Detailed technical information

2. **Structured Logging**
   - Always use structured fields
   - Include correlation IDs
   - Add contextual information
   - Avoid logging sensitive data

3. **Performance**
   - Use async logging in production
   - Configure appropriate buffer sizes
   - Implement log sampling for high-volume events
   - Monitor logging overhead

4. **Security**
   - Never log passwords, tokens, or PII
   - Implement log filtering
   - Use encryption for log transport
   - Control access to log systems

5. **Retention**
   - Define retention policies
   - Archive important logs
   - Delete old logs automatically
   - Consider compliance requirements