source_file
    ::= { use_directive } { top_level_declaration }

use_directive
    ::= "use" package_path ";"

package_path
    ::= identifier { "::" identifier }

top_level_declaration
    ::= data_decl
      | interface_decl
      | extension_decl
      | guard_decl
      | type_alias_decl
      | const_decl
      | var_decl

data_decl
    ::= type_identifier  [ "<" type_params ">" ] [ ":" interfaces ] [ ":::" tag_type ]
        ( "{" { member } "}" | "(" [ short_field_list ] ")" ";" )

interfaces
    ::= type_identifier { "," type_identifier }

tag_type
    ::= type | "none"

member
    ::= variant_decl
      | field_decl
      | unsafe_variant_block
      | const_decl
      | property_decl
      | method_decl

unsafe_variant_block
    ::= "unsafe" "{" { variant_decl } "}"

variant_decl
    ::= [ "unsafe" ] "::" public_id [ "(" payload_field_list ")" ] [ "=" expression ] [ ":::" expression ] ";"

payload_field_list
    ::= payload_field { "," payload_field }

payload_field
    ::= [ field_modifier ] [ type_identifier ":" ] type [ "=" expression ] [ ":::" decimal_literal ]

short_field_list
    ::= short_field { "," short_field }

short_field
    ::= [ field_modifier ] ( type_identifier | "_" ) ":" type [ field_storage_modifier ] [ "=" expression ] [ ":::" decimal_literal ]

field_storage_modifier
    ::= "embed" | "static" | "astatic" | "gstatic"

field_decl
    ::= short_field ";"

field_modifier
    ::= "$" | "#" | "#$"

interface_decl
    ::= "~" type_identifier [ "<" type_params ">" ] "{" { interface_member } "}"

interface_member
    ::= interface_prop_decl
      | method_decl
      | guard_decl

interface_prop_decl
    ::= type_identifier [ "?" ] ":" type ( ";" | "=>" expression ";" | block )

extension_decl
    ::= "+" type "{" { member } "}"

guard_decl
    ::= "^" type_identifier [ "<" type_params ">" ] "=" target_type [ ":" parent_list ] "must" expression ";"

target_type
    ::= "@" | type

parent_list
    ::= guard_parent { "," guard_parent }

guard_parent
    ::= "^" type_identifier [ "<" type_args ">" ]

type_alias_decl
    ::= type_identifier [ "<" type_params ">" ] "=" type ";"

type_identifier
    ::= public_id
      | internal_id
      | private_id

const_decl
    ::= identifier [ ":" type ] "::" expression ";"

var_decl
    ::= local_id [ ":" type [ static_modifier ] ] ( "=" | "=^" | ":=" ) ( expression | cascade_chain ) [ "strict" ] ";"

static_modifier
    ::= "static" | "astatic" | "gstatic"

method_decl
    ::= type_identifier [ "?" ] [ "<" type_params ">" ] "(" [ receiver ] [ parameter_list ] ")" { method_modifier } [ "->" return_type ] ( body | ";" )

receiver
    ::= [ "ref" ] "@" [ receiver_constraint ] [ "," ]

receiver_constraint
    ::= "::" public_id
      | "^" type_identifier [ "<" type_args ">" ]

parameter_list
    ::= parameter { "," parameter }

parameter
    ::= [ "ref" ] local_id ":" [ "..." ] type [ "=" expression ]

return_type
    ::= type
      | "@" [ receiver_constraint ]

body
    ::= block | "=>" expression ";"

method_modifier
    ::= "throws" | "unsafe"

property_decl
    ::= type_identifier ":" type ( "=>" expression ";" | block )

type
    ::= primitive_type
      | collection_type
      | tuple_type
      | method_type
      | guarded_type
      | option_type
      | weak_type
      | type_path

option_type
    ::= type "?"

weak_type
    ::= type "weak"

guarded_type
    ::= type "^" type_identifier [ "<" type_args ">" ]

primitive_type
    ::= "Int" | "Uint" | "Long" | "Ulong"
      | "Size" | "Usize" | "Byte"
      | "Float" | "Double"
      | "Bool" | "Char" | "Rune"
      | "String" | "RuneString"

collection_type
    ::= "[" type ( "->" | "->!" | "->?" ) type "]"
      | "[" [ map_key ] [ col_modifier ] type [ ":::" ( decimal_literal | identifier ) ] "]"

map_key
    ::= type "$"

col_modifier
    ::= "+" | "$" | "%" | "+%"

tuple_type
    ::= "(" tuple_type_elem "," tuple_type_elem { "," tuple_type_elem } ")"

tuple_type_elem
    ::= [ public_id ":" ] type

method_type
    ::= "fn" "(" [ method_type_list ] ")" [ "->" type ]

method_type_list
    ::= method_param_type { "," method_param_type }

method_param_type
    ::= [ "ref" ] type

type_params
    ::= type_parameter { "," type_parameter }

type_parameter
    ::= type_identifier [ ":" constraint_list ]

constraint_list
    ::= type_identifier { "," type_identifier }

type_args
    ::= type { "," type }

type_path
    ::= [ "~" ] type_identifier [ "<" type_args ">" ] { "::" [ "~" ] type_identifier [ "<" type_args ">" ] }

statement
    ::= var_decl
      | const_decl
      | destructure_stmt
      | must_stmt
      | may_stmt
      | stage_expr
      | stage_expand_stmt
      | out_stmt
      | match_expr
      | try_catch_stmt
      | try_force_block
      | undo_stmt
      | concurrent_expr
      | spawn_stmt
      | break_stmt
      | continue_stmt
      | fall_stmt
      | return_stmt
      | throw_stmt
      | cascade_stmt
      | expression_stmt

block
    ::= "{" { statement } "}"

break_stmt
    ::= "break" ";"

continue_stmt
    ::= "continue" ";"

return_stmt
    ::= "return" [ expression | cascade_chain ] ";"

cascade_stmt
    ::= [ assignable_path assignment_operator ] cascade_chain [ "strict" ] ";"

cascade_chain
    ::= postfix_expr ".." type_identifier method_invocation { ".." type_identifier method_invocation }

undo_stmt
    ::= "undo" ( block | "=>" statement )

throw_stmt
    ::= "throw" [ expression ] ";"

fall_stmt
    ::= "fall" ";"

destructure_stmt
    ::= destructure_pattern ( "=" | ":=" ) expression ";"

destructure_pattern
    ::= "(" destructure_target "," destructure_target { "," destructure_target } ")"

destructure_target
    ::= local_id | "_"

expression_stmt
    ::= expression [ "strict" ] ";"

must_stmt
    ::= "must" ( condition ";" | condition_block )
      | "must" ( condition | condition_block ) "or" ( block | "=>" statement | statement )
      | "must!" ( condition ";" | condition_block )

may_stmt
    ::= "may" condition ( block | "=>" statement ) [ "or" may_tail ]

may_tail
    ::= "may" condition ( block | "=>" statement ) [ "or" may_tail ]
      | block
      | "=>" statement
      | statement

condition_block
    ::= "{" condition ";" { condition ";" } "}"

condition
    ::= condition_clause { "and" condition_clause }

condition_clause
    ::= conditional_binding
      | conditional_assignment
      | expression

conditional_binding
    ::= local_id ":=" prefix_control_expr

conditional_assignment
    ::= assignable_path ( "=" | "=^" ) prefix_control_expr

assignable_path
    ::= [ "@" "." ] identifier { "." type_identifier }

stage_expr
    ::= [ stage_header ] "in" [ iterator ] [ "by" type_identifier ] ( block | "=>" statement )

stage_header
    ::= [ "ref" ] local_id
      | destructure_pattern

stage_expand_stmt
    ::= stage_header "in" iterator [ "by" type_identifier ] ";"

iterator
    ::= range_expr | expression

range_expr
    ::= expression ".." expression [ ".." expression ]

out_stmt
    ::= "out" "count" [ local_id [ ":" type ] ] [ "must" condition ] ";"
      | "out" [ out_reducer ] local_id ( ":" type "=" | ":=" ) expression [ "must" condition ] ";"
      | "out" [ out_reducer ] expression [ "must" condition ] ";"

out_reducer
    ::= "sum" | "min" | "max" | "all" | "any" | "first" | "last"

try_catch_stmt
    ::= "try" ( block | "=>" statement ) catch_arm { catch_arm }

catch_arm
    ::= "catch" [ local_id ":" type ] ( block | "=>" statement )

try_force_block
    ::= "try!" block

concurrent_expr
    ::= "concurrent" [ "throws" ] block

spawn_stmt
    ::= "spawn" [ "by" expression ] ( block | "=>" statement ) { catch_arm }
        [ "rollback" [ "by" expression ] ( block | "=>" statement ) ]

match_expr
    ::= "match" [ "!" | "?" ] [ expression ] "{" match_arm { match_arm } "}"

match_arm
    ::= pattern { ( "or" | "," ) pattern } [ "must" condition ] ( block | "=>" expression [ ";" ] )
      | condition ( block | "=>" expression [ ";" ] )

pattern
    ::= variant_pattern
      | guard_pattern
      | literal
      | "_"

variant_pattern
    ::= "." type_identifier [ "(" pattern_binding_list ")" ]

guard_pattern
    ::= "^" type_identifier [ "<" type_args ">" ]

pattern_binding_list
    ::= local_id { "," local_id }

expression
    ::= assignment_expr

assignment_expr
    ::= coalescing_expr [ assignment_operator assignment_expr ]

coalescing_expr
    ::= logical_or_expr [ "??" coalescing_expr ]

assignment_operator
    ::= "=" | ":=" | "+=" | "-=" | "*=" | "/=" | "%="
      | "=^" | "+=^" | "-=^" | "*=^" | "/=^" | "%=^"
      | "=::"

logical_or_expr
    ::= logical_and_expr { "or" logical_and_expr }

logical_and_expr
    ::= prefix_control_expr { "and" prefix_control_expr }

prefix_control_expr
    ::= { prefix_control_operator } relational_expr

prefix_control_operator
    ::= "not" | "try?" | "try!"

relational_expr
    ::= bitwise_expr { relational_operator bitwise_expr }

relational_operator
    ::= "<" | ">" | "<=" | ">=" | "==" | "!="

bitwise_expr
    ::= shift_expr { bitwise_operator shift_expr }

bitwise_operator
    ::= "band" | "bor" | "bxor"

shift_expr
    ::= additive_expr { shift_operator additive_expr }

shift_operator
    ::= "bsl" | "bsr"

additive_expr
    ::= multiplicative_expr { additive_operator multiplicative_expr }

additive_operator
    ::= "+" | "-"

multiplicative_expr
    ::= type_op_expr { multiplicative_operator type_op_expr }

multiplicative_operator
    ::= "*" | "/" | "%"

type_op_expr
    ::= unary_expr { type_operator }

type_operator
    ::= ( "as" | "as?" | "as!" | "unas" ) ( type | "." type_identifier | "^" )
      | "is" [ "not" ] ( type | variant_pattern | guard_pattern )

unary_expr
    ::= { unary_operator } postfix_expr

unary_operator
    ::= "bnot" | "-" | "+"

postfix_expr
    ::= primary_expr { postfix_operation }

postfix_operation
    ::= method_invocation
      | member_access
      | index_access
      | force_unwrap
      | copy_update_suffix

method_invocation
    ::= [ "?" | "!" ] [ "<" type_args ">" ] "(" [ argument_list ] ")"

member_access
    ::= ( "." | "?." ) type_identifier [ "?" | "!" ]

index_access
    ::= [ "?" ] "[" expr_list "]"

force_unwrap
    ::= "!"

copy_update_suffix
    ::= "with" ( variant_update [ copy_update_body ] | copy_update_body )

variant_update
    ::= "::" public_id [ variant_payload_init_body ]

primary_expr
    ::= literal
      | contextual_expr
      | lambda_expr
      | match_expr
      | stage_expr
      | concurrent_expr
      | try_force_construct
      | tuple_expr
      | collection_init
      | struct_init_expr
      | identifier
      | "(" expression ")"

contextual_expr
    ::= "." type_identifier [ "<" type_args ">" ] [ "(" [ argument_list ] ")" ] [ "{" [ initializer_list ] "}" ]

lambda_expr
    ::= lambda_params "=>" ( expression | block )

lambda_params
    ::= local_id | "(" [ lambda_param_list ] ")"

lambda_param_list
    ::= lambda_param { "," lambda_param }

lambda_param
    ::= [ "ref" ] local_id [ ":" type ]

try_force_construct
    ::= "try!" ( expression | block )

tuple_expr
    ::= "(" tuple_expr_elem "," tuple_expr_elem { "," tuple_expr_elem } ")"

tuple_expr_elem
    ::= [ public_id ":" ] expression

collection_init
    ::= collection_type ( "{" [ collection_elem_list ] "}" | ".New(" [ argument_list ] ")" )
      | "{" [ collection_elem_list ] "}"

collection_elem_list
    ::= collection_elem { "," collection_elem } [ "," ]

collection_elem
    ::= [ collection_elem_key "=" ] coalescing_expr

collection_elem_key
    ::= logical_or_expr | "_"

expr_list
    ::= expression { "," expression } [ "," ]

struct_init_expr
    ::= type_path [ variant_payload_init_body ] [ struct_init_body ] [ init_modifier ]

variant_payload_init_body
    ::= "(" [ argument_list ] ")"

struct_init_body
    ::= "{" [ field_init_list ] "}"

field_init_list
    ::= field_init { "," field_init } [ "," ]

field_init
    ::= named_field_init | positional_field_init

named_field_init
    ::= type_identifier ":" expression

positional_field_init
    ::= expression

init_modifier
    ::= "zeroed" | "noinit"

copy_update_body
    ::= "{" initializer_list "}"

initializer_list
    ::= field_assignment { "," field_assignment } [ "," ]

field_assignment
    ::= type_identifier ":" expression

argument_list
    ::= argument { "," argument } [ "," ]

argument
    ::= [ ( local_id | type_identifier ) ":" ] [ "ref" ] expression

identifier
    ::= public_id
      | internal_id
      | private_id
      | local_id
      | escaped_keyword

escaped_keyword
    ::= "`" keyword "`"

public_id
    ::= uppercase_letter { alphanumeric } (* MUST NOT terminate with "_" *)

internal_id
    ::= uppercase_letter { alphanumeric } "_"

private_id
    ::= "_" lowercase_letter { alphanumeric }

local_id
    ::= lowercase_letter { alphanumeric }

literal
    ::= integer_literal
      | float_literal
      | bool_literal
      | char_literal
      | string_literal
      | raw_string_literal
      | "none"

integer_literal
    ::= ( decimal_literal | hex_literal | binary_literal ) [ int_suffix ]

decimal_literal
    ::= digit { [ "_" ] digit }

hex_literal
    ::= "0x" hex_digit { [ "_" ] hex_digit }

binary_literal
    ::= "0b" binary_digit { [ "_" ] binary_digit }

int_suffix
    ::= "u" | "L" | "uL" | "b"

float_literal
    ::= decimal_literal "." decimal_literal [ exponent ] [ float_suffix ]

exponent
    ::= ( "e" | "E" ) [ "+" | "-" ] decimal_literal

float_suffix
    ::= "f" | "d"

bool_literal
    ::= "true" | "false"

char_literal
    ::= "'" ( source_char_except_single_quote | escape_sequence ) "'"

string_literal
    ::= '"' { source_char_except_double_quote | escape_sequence | interpolated_segment } '"'

interpolated_segment
    ::= "\(" expression ")"

raw_string_literal
    ::= '"""' { raw_char } '"""'

raw_char
    ::= source_char | new_line

escape_sequence
    ::= "\" ( simple_escape | numeric_escape )

simple_escape
    ::= "n" | "r" | "t" | "\" | '"' | "'" | "0"

numeric_escape
    ::= "u" hex_digit hex_digit hex_digit hex_digit
      | "U" hex_digit hex_digit hex_digit hex_digit hex_digit hex_digit hex_digit hex_digit
      | "x" hex_digit hex_digit

comment
    ::= line_comment | block_comment

line_comment
    ::= "//" { source_char_except_new_line }

block_comment
    ::= "/*" { block_char | block_comment } "*/"

whitespace
    ::= space | tab | new_line

space
    ::= " "

tab
    ::= "\t"

new_line
    ::= "\n" | "\r" | "\r\n"

uppercase_letter
    ::= "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J"
      | "K" | "L" | "M" | "N" | "O" | "P" | "Q" | "R" | "S" | "T"
      | "U" | "V" | "W" | "X" | "Y" | "Z"

lowercase_letter
    ::= "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j"
      | "k" | "l" | "m" | "n" | "o" | "p" | "q" | "r" | "s" | "t"
      | "u" | "v" | "w" | "x" | "y" | "z"

digit
    ::= "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"

hex_digit
    ::= digit
      | "a" | "b" | "c" | "d" | "e" | "f"
      | "A" | "B" | "C" | "D" | "E" | "F"

binary_digit
    ::= "0" | "1"

alphanumeric
    ::= uppercase_letter | lowercase_letter | digit

keyword
    ::= "concurrent" | "spawn" | "rollback" | "throws" | "unsafe"
      | "must", "may" | "in" | "by" | "out" | "break" | "continue"
      | "match" | "fall" | "try" | "catch" | "throw" | "undo"
      | "return" | "use" | "with" | "ref" | "weak"
      | "embed" | "static" | "astatic" | "gstatic"
      | "zeroed" | "noinit" | "strict" | "none" | "fn"
      | "as" | "unas" | "is" | "and" | "or" | "not"
      | "band" | "bor" | "bxor" | "bnot" | "bsl" | "bsr"

source_char
    ::= (* Any valid UTF-8 character *)

source_char_except_single_quote
    ::= (* Any valid UTF-8 character except "'" and "\" *)

source_char_except_double_quote
    ::= (* Any valid UTF-8 character except '"' and "\" *)

source_char_except_new_line
    ::= (* Any valid UTF-8 character except line terminators *)

block_char
    ::= (* Any sequence of UTF-8 characters not containing "/*" or "*/" *)