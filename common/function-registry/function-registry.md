我们现在来看下apache impala根目录common下的function-registry文件下的内容：其中.gitignore内容如下：

```ignore
*.pyc
generated_functions.py
```

其中CMakeLists.txt内容如下：

```cmake
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

set(BE_OUTPUT_DIR ${CMAKE_SOURCE_DIR}/be/generated-sources)
set(FE_OUTPUT_DIR ${CMAKE_SOURCE_DIR}/fe/generated-sources/gen-java/org/apache/impala/)

# Set output files for dependency tracking
set(CODE_GEN_OUTPUT
  ${FE_OUTPUT_DIR}/builtins/ScalarBuiltins.java
)

set(GEOSPATIAL_CODE_GEN_OUTPUT
  ${FE_OUTPUT_DIR}/builtins/ST_ConvexHull_Wrapper.java
  ${FE_OUTPUT_DIR}/builtins/ST_LineString_Wrapper.java
  ${FE_OUTPUT_DIR}/builtins/ST_MultiPoint_Wrapper.java
  ${FE_OUTPUT_DIR}/builtins/ST_Polygon_Wrapper.java
  ${FE_OUTPUT_DIR}/builtins/ST_Union_Wrapper.java
)

# Source python files
set(FUNCTION_REGISTRY_INPUT
  gen_builtins_catalog.py
  impala_functions.py
)

set(GEOSPATIAL_WRAPPER_INPUT
  gen_builtins_catalog.py
  gen_geospatial_udf_wrappers.py
)

# Run the python scripts
add_custom_command(
  OUTPUT ${CODE_GEN_OUTPUT}
  COMMAND ./gen_builtins_catalog.py
  DEPENDS ${FUNCTION_REGISTRY_INPUT}
  COMMENT "Generating files for builtins."
  VERBATIM
)

add_custom_command(
  OUTPUT ${GEOSPATIAL_CODE_GEN_OUTPUT}
  COMMAND ./gen_geospatial_udf_wrappers.py
  DEPENDS ${GEOSPATIAL_WRAPPER_INPUT}
  COMMENT "Generating UDF wrappers for geospatial builtins."
  VERBATIM
)

add_custom_target(function-registry ALL DEPENDS ${CODE_GEN_OUTPUT})

if($ENV{USE_APACHE_HIVE_3} STREQUAL "false")
  add_custom_target(geospatial-udf-wrappers ALL DEPENDS ${GEOSPATIAL_CODE_GEN_OUTPUT})
  add_dependencies(geospatial-udf-wrappers function-registry)
else()
  # Empty target to skip wrapper generation
  add_custom_target(geospatial-udf-wrappers)
endif()
```

其中gen_buildins_catagory.py内容如下：

```python
#!/usr/bin/env python3
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

# This script generates the FE calls to populate the builtins.
# To add a builtin, add an entry to impala_functions.py.

from __future__ import absolute_import, division, print_function
import os
import impala_functions

java_registry_preamble = '\
// \n\
//  Licensed under the Apache License, Version 2.0 (the "License");\n\
//  you may not use this file except in compliance with the License.\n\
//  You may obtain a copy of the License at\n\
// \n\
//  http://www.apache.org/licenses/LICENSE-2.0\n\
// \n\
//  Unless required by applicable law or agreed to in writing, software\n\
//  distributed under the License is distributed on an "AS IS" BASIS,\n\
//  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.\n\
//  See the License for the specific language governing permissions and\n\
//  limitations under the License.\n\
\n\
// This is a generated file, DO NOT EDIT.\n\
// To add new functions, see the generator at\n\
// common/function-registry/gen_builtins_catalog.py or the function list at\n\
// common/function-registry/impala_functions.py.\n\
\n\
package org.apache.impala.builtins;\n\
\n\
import org.apache.impala.catalog.Type;\n\
import org.apache.impala.catalog.Db;\n\
\n\
public class ScalarBuiltins { \n\
  public static void initBuiltins(Db db) { \
\n'

java_registry_epilogue = '\
  }\n\
}\n'

FE_PATH = os.path.expandvars(
  "$IMPALA_HOME/fe/generated-sources/gen-java/org/apache/impala/builtins/")

# This contains all the metadata to describe all the builtins.
# Each meta data entry is itself a map to store all the meta data
#   - fn_name, ret_type, args, symbol, sql_names
meta_data_entries = []


# Read in the function and add it to the meta_data_entries map
def add_function(fn_meta_data, user_visible):
  assert 4 <= len(fn_meta_data) <= 6, \
         "Invalid function entry in impala_functions.py:\n\t" + repr(fn_meta_data)
  entry = {}
  entry["sql_names"] = fn_meta_data[0]
  entry["ret_type"] = fn_meta_data[1]
  entry["args"] = fn_meta_data[2]
  entry["symbol"] = fn_meta_data[3]
  if len(fn_meta_data) >= 5:
    entry["prepare"] = fn_meta_data[4]
  if len(fn_meta_data) >= 6:
    entry["close"] = fn_meta_data[5]
  entry["user_visible"] = user_visible
  meta_data_entries.append(entry)


def generate_fe_entry(entry, name):
  java_output = ""
  java_output += "\"" + name + "\""
  java_output += ", \"" + entry["symbol"] + "\""
  if entry["user_visible"]:
    java_output += ", true"
  else:
    java_output += ", false"

  if 'prepare' in entry:
    java_output += ', "%s"' % entry["prepare"]
    if 'close' in entry:
      java_output += ', "%s"' % entry["close"]
    else:
      java_output += ', null'

  # Check the last entry for varargs indicator.
  if entry["args"] and entry["args"][-1] == "...":
    entry["args"].pop()
    java_output += ", true"
  else:
    java_output += ", false"

  java_output += ", Type." + entry["ret_type"]
  for arg in entry["args"]:
    java_output += ", Type." + arg
  return java_output


# Generates the FE builtins init file that registers all the builtins.
def generate_fe_registry_init(filename):
  java_registry_file = open(filename, "w")
  java_registry_file.write(java_registry_preamble)

  for entry in meta_data_entries:
    for name in entry["sql_names"]:
      java_output = generate_fe_entry(entry, name)
      java_registry_file.write("    db.addScalarBuiltin(%s);\n" % java_output)

  java_registry_file.write("\n")
  java_registry_file.write(java_registry_epilogue)
  java_registry_file.close()


if __name__ == "__main__":
  # Read the function metadata inputs
  for function in impala_functions.visible_functions:
    add_function(function, True)
  for function in impala_functions.invisible_functions:
    add_function(function, False)

  if not os.path.exists(FE_PATH):
    os.makedirs(FE_PATH)

  generate_fe_registry_init(FE_PATH + "ScalarBuiltins.java")
```

其中gen_geospatial_udf_wrappers.py内容如下：

```python
#!/usr/bin/env python3
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.


# Java wrapper class generator for Hive ESRI varargs UDFs to bridge the varargs support
# in Impala. A generated class is extending the original UDF and adding wrapper
# 'evaluate' methods projecting the varargs method as an n parameter method.

from __future__ import absolute_import, division, print_function
import os
from gen_builtins_catalog import FE_PATH

LICENSE = """
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.

// This is a generated file, DO NOT EDIT.
// To add new functions, see the generator at
// common/function-registry/gen_geospatial_builtins_wrappers.py"""


ARGUMENT_EXCEPTION = "UDFArgumentException"
ARGUMENT_LENGTH_EXCEPTION = "UDFArgumentLengthException"
UDF_PACKAGE = "org.apache.hadoop.hive.ql.udf.esri"
DOUBLE_TYPE = "org.apache.hadoop.hive.serde2.io.DoubleWritable"
BYTE_TYPE = "org.apache.hadoop.io.BytesWritable"


class Wrapper():

  METHOD_FORMAT = ("""  public {return_type} evaluate({parameter_list}) {exception_clause}{{
    return super.evaluate({argument_list});
  }}""")

  CLASS_FORMAT = """public class {class_name} extends {base_class} {{
{methods}
}}"""

  FILE_FORMAT = """{license}

package {package};

{wrapper_class}"""

  EXCEPTION_CLAUSE_FORMAT = "throws org.apache.hadoop.hive.ql.exec.%s"

  def __init__(self, original_class, parameter_type, parameter_range, throws=None):
    self.original_class = original_class
    self.parameter_type = parameter_type
    self.throws = throws
    self.parameter_range = parameter_range
    # Return type is always BytesWritable with the current cases
    self.return_type = BYTE_TYPE

  def generate_parameter(self, num):
    return "{parameter_type} arg{num}".format(
      parameter_type=self.parameter_type, num=num
    )

  def generate_argument(self, num):
    return "arg%d" % num

  def generate_argument_list(self, num):
    arguments = list()
    for i in range(num):
      arguments.append(self.generate_argument(i))
    return ", ".join(arguments)

  def generate_parameter_list(self, num):
    parameters = list()
    for i in range(num):
      parameters.append(self.generate_parameter(i))
    return ", ".join(parameters)

  def generate_method(self, num):
    exception_clause = ""
    if self.throws:
      exception_clause = self.EXCEPTION_CLAUSE_FORMAT % self.throws
    return self.METHOD_FORMAT.format(
      return_type=self.return_type,
      parameter_list=self.generate_parameter_list(num),
      exception_clause=exception_clause,
      argument_list=self.generate_argument_list(num),
    )

  def generate_methods(self):
    methods = list()

    for i in self.parameter_range:
      methods.append(self.generate_method(i))

    return "\n\n".join(methods)

  def generate_wrapper_class(self):
    return self.CLASS_FORMAT.format(
      class_name=self.generate_wrapper_name(),
      base_class=self.original_class,
      methods=self.generate_methods()
    )

  def generate_file(self):
    return self.FILE_FORMAT.format(
      license=LICENSE,
      package="org.apache.impala.builtins",
      wrapper_class=self.generate_wrapper_class()
    )

  def generate_wrapper_name(self):
    return "%s_Wrapper" % self.original_class.split('.').pop()

  def get_filename(self):
    return "%s.java" % self.generate_wrapper_name()


WRAPPERS = [Wrapper("%s.ST_ConvexHull" % UDF_PACKAGE, BYTE_TYPE, list(range(1, 9, 1))),
            Wrapper("%s.ST_LineString" % UDF_PACKAGE, DOUBLE_TYPE, list(range(2, 15, 2)),
              ARGUMENT_EXCEPTION),
            Wrapper("%s.ST_MultiPoint" % UDF_PACKAGE, DOUBLE_TYPE, list(range(2, 15, 2)),
              ARGUMENT_LENGTH_EXCEPTION),
            Wrapper("%s.ST_Polygon" % UDF_PACKAGE, DOUBLE_TYPE, list(range(6, 15, 2)),
              ARGUMENT_LENGTH_EXCEPTION),
            Wrapper("%s.ST_Union" % UDF_PACKAGE, BYTE_TYPE, list(range(2, 15, 1)))]

if __name__ == "__main__":
  if not os.path.exists(FE_PATH):
    os.makedirs(FE_PATH)

  for wrapper_config in WRAPPERS:
    path = os.path.join(FE_PATH, wrapper_config.get_filename())
    wrapper_class_file = open(path, "w")
    wrapper_class_file.write(wrapper_config.generate_file())
    wrapper_class_file.close()
```

其中impala_funcitons.py内容如下：# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements. See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership. The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied. See the License for the
# specific language governing permissions and limitations
# under the License.
# This is a list of all the functions that are not auto-generated.
# It contains all the meta data that describes the function.
templated_type_symbol_map = {
  'bool' : 'b',
  'int8_t' : 'a',
  'int16_t' : 's',
  'int32_t' : 'i',
  'int64_t' : 'l',
  'float' : 'f',
  'double' : 'd',
  'string' : 'NS_11StringValueE',
  'timestamp' : 'NS_14TimestampValueE'
}
# Generates the BE symbol for the Compute Function class_name::fn_name<templated_type>.
# Does not handle varargs.
# TODO: this is a stopgap. ComputeFunctions are being removed and we can use the
# symbol lookup code in the BE.
def symbol(class_name, fn_name, templated_type = None):
  sym = '_ZN6impala'
  sym += str(len(class_name)) + class_name
  sym += str(len(fn_name)) + fn_name
  if templated_type == None:
    sym += 'EPNS_4ExprEPNS_8TupleRowE'
  else:
    sym += 'I'
    sym += templated_type_symbol_map[templated_type]
    sym += 'EEPvPNS_4ExprEPNS_8TupleRowE'
  return sym
# The format is:
# [sql aliases], <return_type>, [<args>], <backend symbol>,
# With an optional
# <prepare symbol>, <close symbol>
#
# 'sql aliases' are the function names that can be used from sql. There must be at least
# one per function.
#
# The symbol can be empty for functions that are not yet implemented or are special-cased
# in Expr::CreateExpr() (i.e., functions that are implemented via a custom Expr class
# rather than a single function).
visible_functions = [
  [['udf_pi'], 'DOUBLE', [], 'impala::UdfBuiltins::Pi'],
  [['udf_abs'], 'DOUBLE', ['DOUBLE'], 'impala::UdfBuiltins::Abs'],
  [['udf_lower'], 'STRING', ['STRING'], 'impala::UdfBuiltins::Lower'],
  [['max_int'], 'INT', [],
   '_ZN6impala11UdfBuiltins6MaxIntEPN10impala_udf15FunctionContextE'],
  [['max_tinyint'], 'TINYINT', [],
   '_ZN6impala11UdfBuiltins10MaxTinyIntEPN10impala_udf15FunctionContextE'],
  [['max_smallint'], 'SMALLINT', [],
   '_ZN6impala11UdfBuiltins11MaxSmallIntEPN10impala_udf15FunctionContextE'],
  [['max_bigint'], 'BIGINT', [],
   '_ZN6impala11UdfBuiltins9MaxBigIntEPN10impala_udf15FunctionContextE'],
  [['min_int'], 'INT', [],
   '_ZN6impala11UdfBuiltins6MinIntEPN10impala_udf15FunctionContextE'],
  [['min_tinyint'], 'TINYINT', [],
   '_ZN6impala11UdfBuiltins10MinTinyIntEPN10impala_udf15FunctionContextE'],
  [['min_smallint'], 'SMALLINT', [],
   '_ZN6impala11UdfBuiltins11MinSmallIntEPN10impala_udf15FunctionContextE'],
  [['min_bigint'], 'BIGINT', [],
   '_ZN6impala11UdfBuiltins9MinBigIntEPN10impala_udf15FunctionContextE'],
  [['is_nan'], 'BOOLEAN', ['DOUBLE'],
   '_ZN6impala11UdfBuiltins5IsNanEPN10impala_udf15FunctionContextERKNS1_9DoubleValE'],
  [['is_inf'], 'BOOLEAN', ['DOUBLE'],
   '_ZN6impala11UdfBuiltins5IsInfEPN10impala_udf15FunctionContextERKNS1_9DoubleValE'],
  [['trunc'], 'TIMESTAMP', ['TIMESTAMP', 'STRING'],
   '_ZN6impala11UdfBuiltins17TruncForTimestampEPN10impala_udf15FunctionContextERKNS1_12TimestampValERKNS1_9StringValE',
   '_ZN6impala11UdfBuiltins24TruncForTimestampPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins22TruncForTimestampCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['trunc'], 'DATE', ['DATE', 'STRING'],
   '_ZN6impala11UdfBuiltins12TruncForDateEPN10impala_udf15FunctionContextERKNS1_7DateValERKNS1_9StringValE',
   '_ZN6impala11UdfBuiltins19TruncForDatePrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins17TruncForDateCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  # Don't add an entry for EXTRACT(STRING, TIMESTAMP). STRINGs may be used to represent
  # TIMESTAMPs meaning EXTRACT(STRING, STRING) is valid. If EXTRACT(STRING, TIMESTAMP)
  # is added, it takes precedence over the existing EXTRACT(TIMESTAMP, STRING)
  # which could break users.
  [['extract'], 'BIGINT', ['TIMESTAMP', 'STRING'],
   '_ZN6impala11UdfBuiltins19ExtractForTimestampEPN10impala_udf15FunctionContextERKNS1_12TimestampValERKNS1_9StringValE',
   '_ZN6impala11UdfBuiltins26ExtractForTimestampPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins24ExtractForTimestampCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['date_part'], 'BIGINT', ['STRING', 'TIMESTAMP'],
   '_ZN6impala11UdfBuiltins20DatePartForTimestampEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_12TimestampValE',
   '_ZN6impala11UdfBuiltins27DatePartForTimestampPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins25DatePartForTimestampCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['extract'], 'BIGINT', ['DATE', 'STRING'],
   '_ZN6impala11UdfBuiltins14ExtractForDateEPN10impala_udf15FunctionContextERKNS1_7DateValERKNS1_9StringValE',
   '_ZN6impala11UdfBuiltins21ExtractForDatePrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins19ExtractForDateCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['date_part'], 'BIGINT', ['STRING', 'DATE'],
   '_ZN6impala11UdfBuiltins15DatePartForDateEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_7DateValE',
   '_ZN6impala11UdfBuiltins22DatePartForDatePrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins20DatePartForDateCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['madlib_encode_vector'], 'STRING', ['STRING'],
    '_ZN6impala11UdfBuiltins12EncodeVectorEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['madlib_decode_vector'], 'STRING', ['STRING'],
    '_ZN6impala11UdfBuiltins12DecodeVectorEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['madlib_print_vector'], 'STRING', ['STRING'],
    '_ZN6impala11UdfBuiltins11PrintVectorEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['madlib_vector'], 'STRING', ['DOUBLE', '...'],
    '_ZN6impala11UdfBuiltins8ToVectorEPN10impala_udf15FunctionContextEiPKNS1_9DoubleValE'],
  [['madlib_vector_get'], 'DOUBLE', ['BIGINT', 'STRING'],
    '_ZN6impala11UdfBuiltins9VectorGetEPN10impala_udf15FunctionContextERKNS1_9BigIntValERKNS1_9StringValE'],
  # Timestamp functions
  [['monthname'], 'STRING', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions13LongMonthNameEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['next_day'], 'TIMESTAMP', ['TIMESTAMP', 'STRING'], '_ZN6impala18TimestampFunctions7NextDayEPN10impala_udf15FunctionContextERKNS1_12TimestampValERKNS1_9StringValE'],
  [['last_day'], 'TIMESTAMP', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions7LastDayEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['unix_timestamp'], 'BIGINT', ['STRING'], '_ZN6impala18TimestampFunctions14UnixFromStringEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['year'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions4YearEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['quarter'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions7QuarterEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['month'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions5MonthEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['dayofweek'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions9DayOfWeekEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['day', 'dayofmonth'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions10DayOfMonthEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['dayofyear'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions9DayOfYearEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['week', 'weekofyear'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions10WeekOfYearEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['hour'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions4HourEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['hour'], 'INT', ['STRING'], '_ZN6impala18TimestampFunctions4HourEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['minute'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions6MinuteEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['minute'], 'INT', ['STRING'], '_ZN6impala18TimestampFunctions6MinuteEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['second'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions6SecondEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['second'], 'INT', ['STRING'], '_ZN6impala18TimestampFunctions6SecondEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['millisecond'], 'INT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions11MillisecondEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['millisecond'], 'INT', ['STRING'], '_ZN6impala18TimestampFunctions11MillisecondEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['to_date'], 'STRING', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions6ToDateEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['dayname'], 'STRING', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions11LongDayNameEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['date_trunc'], 'TIMESTAMP', ['STRING', 'TIMESTAMP'],
   '_ZN6impala11UdfBuiltins21DateTruncForTimestampEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_12TimestampValE',
   '_ZN6impala11UdfBuiltins28DateTruncForTimestampPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins26DateTruncForTimestampCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['years_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9date_time14years_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['years_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9date_time14years_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['years_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9date_time14years_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['years_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9date_time14years_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_add', 'add_months'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb1EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_add', 'add_months'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb1EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb1EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb1EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['weeks_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9gregorian14weeks_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['weeks_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9gregorian14weeks_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['weeks_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9gregorian14weeks_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['weeks_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9gregorian14weeks_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['days_add', 'date_add', 'adddate'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9gregorian13date_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['days_add', 'date_add', 'adddate'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9gregorian13date_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['days_sub', 'date_sub', 'subdate'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9gregorian13date_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['days_sub', 'date_sub', 'subdate'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9gregorian13date_durationELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['hours_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost10posix_time5hoursELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['hours_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost10posix_time5hoursELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['hours_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost10posix_time5hoursELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['hours_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost10posix_time5hoursELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['minutes_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost10posix_time7minutesELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['minutes_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost10posix_time7minutesELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['minutes_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost10posix_time7minutesELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['minutes_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost10posix_time7minutesELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['seconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost10posix_time7secondsELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['seconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost10posix_time7secondsELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['seconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost10posix_time7secondsELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['seconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost10posix_time7secondsELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKS7_RKT0_'],
  [['milliseconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['milliseconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['milliseconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['milliseconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['microseconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['microseconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['microseconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['microseconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['nanoseconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['nanoseconds_add'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['nanoseconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['nanoseconds_sub'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9date_time18subsecond_durationINS4_10posix_time13time_durationELl1000000000EEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['datediff'], 'INT', ['TIMESTAMP', 'TIMESTAMP'], '_ZN6impala18TimestampFunctions8DateDiffEPN10impala_udf15FunctionContextERKNS1_12TimestampValES6_'],
  [['unix_timestamp'], 'BIGINT', [], '_ZN6impala18TimestampFunctions4UnixEPN10impala_udf15FunctionContextE'],
  [['unix_timestamp'], 'BIGINT', ['TIMESTAMP'], '_ZN6impala18TimestampFunctions4UnixEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['unix_timestamp'], 'BIGINT', ['STRING', 'STRING'], '_ZN6impala18TimestampFunctions4UnixEPN10impala_udf15FunctionContextERKNS1_9StringValES6_',
          '_ZN6impala18TimestampFunctions13ToUnixPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
          '_ZN6impala18TimestampFunctions20UnixAndFromUnixCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['unix_micros_to_utc_timestamp'], 'TIMESTAMP', ['BIGINT'],
      '_ZN6impala18TimestampFunctions24UnixMicrosToUtcTimestampEPN10impala_udf15FunctionContextERKNS1_9BigIntValE'],
  [['utc_to_unix_micros'], 'BIGINT', ['TIMESTAMP'],
    '_ZN6impala18TimestampFunctions15UtcToUnixMicrosEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['from_unixtime'], 'STRING', ['INT'],
      '_ZN6impala18TimestampFunctions8FromUnixIN10impala_udf6IntValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['from_unixtime'], 'STRING', ['INT', 'STRING'],
      '_ZN6impala18TimestampFunctions8FromUnixIN10impala_udf6IntValEEENS2_9StringValEPNS2_15FunctionContextERKT_RKS4_',
      '_ZN6impala18TimestampFunctions15FromUnixPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
     '_ZN6impala18TimestampFunctions20UnixAndFromUnixCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['from_unixtime'], 'STRING', ['BIGINT'],
      '_ZN6impala18TimestampFunctions8FromUnixIN10impala_udf9BigIntValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['from_unixtime'], 'STRING', ['BIGINT', 'STRING'],
      '_ZN6impala18TimestampFunctions8FromUnixIN10impala_udf9BigIntValEEENS2_9StringValEPNS2_15FunctionContextERKT_RKS4_',
      '_ZN6impala18TimestampFunctions15FromUnixPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
      '_ZN6impala18TimestampFunctions20UnixAndFromUnixCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['now', 'current_timestamp'], 'TIMESTAMP', [], '_ZN6impala18TimestampFunctions3NowEPN10impala_udf15FunctionContextE'],
  [['utc_timestamp'], 'TIMESTAMP', [], '_ZN6impala18TimestampFunctions12UtcTimestampEPN10impala_udf15FunctionContextE'],
  [['from_utc_timestamp'], 'TIMESTAMP', ['TIMESTAMP', 'STRING'],
   'impala::TimestampFunctions::FromUtc',
   '_ZN6impala18TimestampFunctions22FromUtcAndToUtcPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala18TimestampFunctions20FromUtcAndToUtcCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['to_utc_timestamp'], 'TIMESTAMP', ['TIMESTAMP', 'STRING'],
   'impala::TimestampFunctions::ToUtc',
   '_ZN6impala18TimestampFunctions22FromUtcAndToUtcPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala18TimestampFunctions20FromUtcAndToUtcCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['timeofday'], 'STRING', [],"impala::TimestampFunctions::TimeOfDay"],
  [['timestamp_cmp'], 'INT', ['TIMESTAMP', 'TIMESTAMP'],
   "impala::TimestampFunctions::TimestampCmp"],
  [['int_months_between'], 'INT', ['TIMESTAMP', 'TIMESTAMP'],
   "impala::TimestampFunctions::IntMonthsBetween"],
  [['months_between'], 'DOUBLE', ['TIMESTAMP', 'TIMESTAMP'],
   "impala::TimestampFunctions::MonthsBetween"],
  [['to_timestamp'], 'TIMESTAMP', ['BIGINT'],
   '_ZN6impala18TimestampFunctions11ToTimestampEPN10impala_udf15FunctionContextERKNS1_9BigIntValE'],
  [['to_timestamp'], 'TIMESTAMP', ['STRING', 'STRING'],
   '_ZN6impala18TimestampFunctions11ToTimestampEPN10impala_udf15FunctionContextERKNS1_9StringValES6_',
   '_ZN6impala18TimestampFunctions13ToUnixPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala18TimestampFunctions20UnixAndFromUnixCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['from_timestamp'], 'STRING', ['TIMESTAMP', 'STRING'],
   'impala::TimestampFunctions::FromTimestamp',
   '_ZN6impala18TimestampFunctions15FromUnixPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala18TimestampFunctions20UnixAndFromUnixCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  # Date functions
  [['monthname'], 'STRING', ['DATE'], '_ZN6impala13DateFunctions13LongMonthNameEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['next_day'], 'DATE', ['DATE', 'STRING'], '_ZN6impala13DateFunctions7NextDayEPN10impala_udf15FunctionContextERKNS1_7DateValERKNS1_9StringValE'],
  [['last_day'], 'DATE', ['DATE'], '_ZN6impala13DateFunctions7LastDayEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['year'], 'INT', ['DATE'], '_ZN6impala13DateFunctions4YearEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['quarter'], 'INT', ['DATE'], '_ZN6impala13DateFunctions7QuarterEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['month'], 'INT', ['DATE'], '_ZN6impala13DateFunctions5MonthEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['dayofweek'], 'INT', ['DATE'], '_ZN6impala13DateFunctions9DayOfWeekEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['day', 'dayofmonth'], 'INT', ['DATE'], '_ZN6impala13DateFunctions10DayOfMonthEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['dayofyear'], 'INT', ['DATE'], '_ZN6impala13DateFunctions9DayOfYearEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['week', 'weekofyear'], 'INT', ['DATE'], '_ZN6impala13DateFunctions10WeekOfYearEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['dayname'], 'STRING', ['DATE'], '_ZN6impala13DateFunctions11LongDayNameEPN10impala_udf15FunctionContextERKNS1_7DateValE'],
  [['date_trunc'], 'DATE', ['STRING', 'DATE'],
   '_ZN6impala11UdfBuiltins16DateTruncForDateEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_7DateValE',
   '_ZN6impala11UdfBuiltins23DateTruncForDatePrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala11UdfBuiltins21DateTruncForDateCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['months_add', 'add_months'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb1EN10impala_udf6IntValELb1EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['months_add', 'add_months'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb1EN10impala_udf9BigIntValELb1EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['months_sub'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb0EN10impala_udf6IntValELb1EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['months_sub'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb0EN10impala_udf9BigIntValELb1EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['years_add'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions11AddSubYearsILb1EN10impala_udf6IntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['years_add'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions11AddSubYearsILb1EN10impala_udf9BigIntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['years_sub'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions11AddSubYearsILb0EN10impala_udf6IntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['years_sub'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions11AddSubYearsILb0EN10impala_udf9BigIntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['weeks_add'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions11AddSubWeeksILb1EN10impala_udf6IntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['weeks_add'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions11AddSubWeeksILb1EN10impala_udf9BigIntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['weeks_sub'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions11AddSubWeeksILb0EN10impala_udf6IntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['weeks_sub'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions11AddSubWeeksILb0EN10impala_udf9BigIntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['days_add', 'date_add', 'adddate'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions10AddSubDaysILb1EN10impala_udf6IntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['days_add', 'date_add', 'adddate'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions10AddSubDaysILb1EN10impala_udf9BigIntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['days_sub', 'date_sub', 'subdate'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions10AddSubDaysILb0EN10impala_udf6IntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['days_sub', 'date_sub', 'subdate'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions10AddSubDaysILb0EN10impala_udf9BigIntValEEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['datediff'], 'INT', ['DATE', 'DATE'], '_ZN6impala13DateFunctions8DateDiffEPN10impala_udf15FunctionContextERKNS1_7DateValES6_'],
  [['current_date'], 'DATE', [], '_ZN6impala13DateFunctions11CurrentDateEPN10impala_udf15FunctionContextE'],
  [['date_cmp'], 'INT', ['DATE', 'DATE'], "_ZN6impala13DateFunctions7DateCmpEPN10impala_udf15FunctionContextERKNS1_7DateValES6_"],
  [['int_months_between'], 'INT', ['DATE', 'DATE'], "_ZN6impala13DateFunctions16IntMonthsBetweenEPN10impala_udf15FunctionContextERKNS1_7DateValES6_"],
  [['months_between'], 'DOUBLE', ['DATE', 'DATE'], "_ZN6impala13DateFunctions13MonthsBetweenEPN10impala_udf15FunctionContextERKNS1_7DateValES6_"],
  # Math builtin functions
  # Add new math builtins that operate on scalar values to
  # org.apache.impala.analysis.FunctionCallExpr::builtinMathScalarFunctionNames_.
  [['pi'], 'DOUBLE', [], 'impala::MathFunctions::Pi'],
  [['e'], 'DOUBLE', [], 'impala::MathFunctions::E'],
  [['abs'], 'BIGINT', ['BIGINT'], 'impala::MathFunctions::Abs'],
  [['abs'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Abs'],
  [['abs'], 'FLOAT', ['FLOAT'], 'impala::MathFunctions::Abs'],
  [['abs'], 'BIGINT', ['INT'], 'impala::MathFunctions::Abs'],
  [['abs'], 'INT', ['SMALLINT'], 'impala::MathFunctions::Abs'],
  [['abs'], 'SMALLINT', ['TINYINT'], 'impala::MathFunctions::Abs'],
  [['sign'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Sign'],
  [['sin'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Sin'],
  [['asin'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Asin'],
  [['cos'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Cos'],
  [['acos'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Acos'],
  [['tan'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Tan'],
  [['atan'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Atan'],
  [['atan2'], 'DOUBLE', ['DOUBLE','DOUBLE'], 'impala::MathFunctions::Atan2'],
  [['cosh'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Cosh'],
  [['tanh'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Tanh'],
  [['sinh'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Sinh'],
  [['cot'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Cot'],
  [['radians'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Radians'],
  [['degrees'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Degrees'],
  [['ceil', 'ceiling', 'dceil'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Ceil'],
  [['floor', 'dfloor'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Floor'],
  [['truncate','dtrunc','trunc'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Truncate'],
  [['round','dround'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Round'],
  [['round','dround'], 'DOUBLE', ['DOUBLE', 'BIGINT'], 'impala::MathFunctions::RoundUpTo'],
  [['exp', 'dexp'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Exp'],
  [['ln','dlog1'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Ln'],
  [['log10','dlog10'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Log10'],
  [['log2'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Log2'],
  [['log'], 'DOUBLE', ['DOUBLE', 'DOUBLE'], 'impala::MathFunctions::Log'],
  [['pow', 'power','dpow','fpow'], 'DOUBLE', ['DOUBLE', 'DOUBLE'], 'impala::MathFunctions::Pow'],
  [['sqrt','dsqrt'], 'DOUBLE', ['DOUBLE'], 'impala::MathFunctions::Sqrt'],
  [['rand','random'], 'DOUBLE', [], 'impala::MathFunctions::Rand',
   '_ZN6impala13MathFunctions11RandPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala13MathFunctions9RandCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['factorial'], 'BIGINT', ['TINYINT'], 'impala::Operators::Factorial_TinyIntVal'],
  [['factorial'], 'BIGINT', ['SMALLINT'], 'impala::Operators::Factorial_SmallIntVal'],
  [['factorial'], 'BIGINT', ['INT'], 'impala::Operators::Factorial_IntVal'],
  [['factorial'], 'BIGINT', ['BIGINT'], 'impala::Operators::Factorial_BigIntVal'],
  [['rand','random'], 'DOUBLE', ['BIGINT'], 'impala::MathFunctions::RandSeed',
   '_ZN6impala13MathFunctions11RandPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala13MathFunctions9RandCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['bin'], 'STRING', ['BIGINT'], 'impala::MathFunctions::Bin'],
  [['hex'], 'STRING', ['BIGINT'], 'impala::MathFunctions::HexInt'],
  [['hex'], 'STRING', ['STRING'], 'impala::MathFunctions::HexString'],
  [['unhex'], 'STRING', ['STRING'], 'impala::MathFunctions::Unhex'],
  [['conv'], 'STRING', ['BIGINT', 'TINYINT', 'TINYINT'],
   'impala::MathFunctions::ConvInt'],
  [['conv'], 'STRING', ['STRING', 'TINYINT', 'TINYINT'],
      'impala::MathFunctions::ConvString'],
  [['pmod'], 'BIGINT', ['BIGINT', 'BIGINT'], 'impala::MathFunctions::PmodBigInt'],
  [['pmod'], 'DOUBLE', ['DOUBLE', 'DOUBLE'], 'impala::MathFunctions::PmodDouble'],
  [['fmod'], 'FLOAT', ['FLOAT', 'FLOAT'], 'impala::MathFunctions::FmodFloat'],
  [['fmod'], 'DOUBLE', ['DOUBLE', 'DOUBLE'], 'impala::MathFunctions::FmodDouble'],
  [['mod'], 'TINYINT', ['TINYINT', 'TINYINT'], 'impala::Operators::Mod_TinyIntVal_TinyIntVal'],
  [['mod'], 'SMALLINT', ['SMALLINT', 'SMALLINT'], 'impala::Operators::Mod_SmallIntVal_SmallIntVal'],
  [['mod'], 'INT', ['INT', 'INT'], 'impala::Operators::Mod_IntVal_IntVal'],
  [['mod'], 'BIGINT', ['BIGINT', 'BIGINT'], 'impala::Operators::Mod_BigIntVal_BigIntVal'],
  [['mod'], 'FLOAT', ['FLOAT', 'FLOAT'], 'impala::MathFunctions::FmodFloat'],
  [['mod'], 'DOUBLE', ['DOUBLE', 'DOUBLE'], 'impala::MathFunctions::FmodDouble'],
  [['mod'], 'DECIMAL', ['DECIMAL', 'DECIMAL'], 'impala::DecimalOperators::Mod_DecimalVal_DecimalVal'],
  [['positive'], 'TINYINT', ['TINYINT'],
   '_ZN6impala13MathFunctions8PositiveIN10impala_udf10TinyIntValEEET_PNS2_15FunctionContextERKS4_'],
  [['positive'], 'SMALLINT', ['SMALLINT'],
   '_ZN6impala13MathFunctions8PositiveIN10impala_udf11SmallIntValEEET_PNS2_15FunctionContextERKS4_'],
  [['positive'], 'INT', ['INT'],
   '_ZN6impala13MathFunctions8PositiveIN10impala_udf6IntValEEET_PNS2_15FunctionContextERKS4_'],
  [['positive'], 'BIGINT', ['BIGINT'],
   '_ZN6impala13MathFunctions8PositiveIN10impala_udf9BigIntValEEET_PNS2_15FunctionContextERKS4_'],
  [['positive'], 'FLOAT', ['FLOAT'],
   '_ZN6impala13MathFunctions8PositiveIN10impala_udf8FloatValEEET_PNS2_15FunctionContextERKS4_'],
  [['positive'], 'DOUBLE', ['DOUBLE'],
   '_ZN6impala13MathFunctions8PositiveIN10impala_udf9DoubleValEEET_PNS2_15FunctionContextERKS4_'],
  [['positive'], 'DECIMAL', ['DECIMAL'],
   '_ZN6impala13MathFunctions8PositiveIN10impala_udf10DecimalValEEET_PNS2_15FunctionContextERKS4_'],
  [['negative'], 'TINYINT', ['TINYINT'],
   '_ZN6impala13MathFunctions8NegativeIN10impala_udf10TinyIntValEEET_PNS2_15FunctionContextERKS4_'],
  [['negative'], 'SMALLINT', ['SMALLINT'],
   '_ZN6impala13MathFunctions8NegativeIN10impala_udf11SmallIntValEEET_PNS2_15FunctionContextERKS4_'],
  [['negative'], 'INT', ['INT'],
   '_ZN6impala13MathFunctions8NegativeIN10impala_udf6IntValEEET_PNS2_15FunctionContextERKS4_'],
  [['negative'], 'BIGINT', ['BIGINT'],
   '_ZN6impala13MathFunctions8NegativeIN10impala_udf9BigIntValEEET_PNS2_15FunctionContextERKS4_'],
  [['negative'], 'FLOAT', ['FLOAT'],
   '_ZN6impala13MathFunctions8NegativeIN10impala_udf8FloatValEEET_PNS2_15FunctionContextERKS4_'],
  [['negative'], 'DOUBLE', ['DOUBLE'],
   '_ZN6impala13MathFunctions8NegativeIN10impala_udf9DoubleValEEET_PNS2_15FunctionContextERKS4_'],
  [['negative'], 'DECIMAL', ['DECIMAL'],
   '_ZN6impala13MathFunctions8NegativeIN10impala_udf10DecimalValEEET_PNS2_15FunctionContextERKS4_'],
  [['quotient'], 'BIGINT', ['BIGINT', 'BIGINT'],
      'impala::MathFunctions::QuotientBigInt'],
  [['quotient'], 'BIGINT', ['DOUBLE', 'DOUBLE'],
      'impala::MathFunctions::QuotientDouble'],
  [['least'], 'TINYINT', ['TINYINT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf10TinyIntValELb1EEET_PNS2_15FunctionContextEiPKS4_'],
  [['least'], 'SMALLINT', ['SMALLINT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf11SmallIntValELb1EEET_PNS2_15FunctionContextEiPKS4_'],
  [['least'], 'INT', ['INT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf6IntValELb1EEET_PNS2_15FunctionContextEiPKS4_'],
  [['least'], 'BIGINT', ['BIGINT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf9BigIntValELb1EEET_PNS2_15FunctionContextEiPKS4_'],
  [['least'], 'FLOAT', ['FLOAT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf8FloatValELb1EEET_PNS2_15FunctionContextEiPKS4_'],
  [['least'], 'DOUBLE', ['DOUBLE', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf9DoubleValELb1EEET_PNS2_15FunctionContextEiPKS4_'],
  [['least'], 'TIMESTAMP', ['TIMESTAMP', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb1EEEN10impala_udf12TimestampValEPNS2_15FunctionContextEiPKS3_'],
  [['least'], 'STRING', ['STRING', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb1EEEN10impala_udf9StringValEPNS2_15FunctionContextEiPKS3_'],
  [['least'], 'DECIMAL', ['DECIMAL', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb1EEEN10impala_udf10DecimalValEPNS2_15FunctionContextEiPKS3_'],
  [['least'], 'DATE', ['DATE', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb1EEEN10impala_udf7DateValEPNS2_15FunctionContextEiPKS3_'],
  [['greatest'], 'TINYINT', ['TINYINT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf10TinyIntValELb0EEET_PNS2_15FunctionContextEiPKS4_'],
  [['greatest'], 'SMALLINT', ['SMALLINT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf11SmallIntValELb0EEET_PNS2_15FunctionContextEiPKS4_'],
  [['greatest'], 'INT', ['INT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf6IntValELb0EEET_PNS2_15FunctionContextEiPKS4_'],
  [['greatest'], 'BIGINT', ['BIGINT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf9BigIntValELb0EEET_PNS2_15FunctionContextEiPKS4_'],
  [['greatest'], 'FLOAT', ['FLOAT', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf8FloatValELb0EEET_PNS2_15FunctionContextEiPKS4_'],
  [['greatest'], 'DOUBLE', ['DOUBLE', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestIN10impala_udf9DoubleValELb0EEET_PNS2_15FunctionContextEiPKS4_'],
  [['greatest'], 'TIMESTAMP', ['TIMESTAMP', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb0EEEN10impala_udf12TimestampValEPNS2_15FunctionContextEiPKS3_'],
  [['greatest'], 'STRING', ['STRING', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb0EEEN10impala_udf9StringValEPNS2_15FunctionContextEiPKS3_'],
  [['greatest'], 'DECIMAL', ['DECIMAL', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb0EEEN10impala_udf10DecimalValEPNS2_15FunctionContextEiPKS3_'],
  [['greatest'], 'DATE', ['DATE', '...'],
   '_ZN6impala13MathFunctions13LeastGreatestILb0EEEN10impala_udf7DateValEPNS2_15FunctionContextEiPKS3_'],
  [['width_bucket'], 'BIGINT', ['DECIMAL', 'DECIMAL', 'DECIMAL', 'INT'],
    '_ZN6impala13MathFunctions11WidthBucketEPN10impala_udf15FunctionContextERKNS1_10DecimalValES6_S6_RKNS1_6IntValE'],
  # Decimal Functions
  # TODO: oracle has decimal support for transcendental functions (e.g. sin()) to very
  # high precisions. Do we need them? It's unclear if other databases do the same.
  [['precision'], 'INT', ['DECIMAL'], 'impala::DecimalFunctions::Precision'],
  [['scale'], 'INT', ['DECIMAL'], 'impala::DecimalFunctions::Scale'],
  [['abs'], 'DECIMAL', ['DECIMAL'], 'impala::DecimalFunctions::Abs'],
  [['ceil', 'ceiling'], 'DECIMAL', ['DECIMAL'], 'impala::DecimalFunctions::Ceil'],
  [['floor','dfloor'], 'DECIMAL', ['DECIMAL'], 'impala::DecimalFunctions::Floor'],
  [['round','dround'], 'DECIMAL', ['DECIMAL'], 'impala::DecimalFunctions::Round'],
  # The reason why we declare the following function several times (instead of simply
  # setting the second argument to BIGINT and relying on implicit casting is because
  # we don't know if the first Decimal argument needs to be cast to double, or the second
  # argument needs to be cast to a BIGINT.
  [['round','dround'], 'DECIMAL', ['DECIMAL', 'TINYINT'], 'impala::DecimalFunctions::RoundTo'],
  [['round','dround'], 'DECIMAL', ['DECIMAL', 'SMALLINT'], 'impala::DecimalFunctions::RoundTo'],
  [['round','dround'], 'DECIMAL', ['DECIMAL', 'INT'], 'impala::DecimalFunctions::RoundTo'],
  [['round','dround'], 'DECIMAL', ['DECIMAL', 'BIGINT'], 'impala::DecimalFunctions::RoundTo'],
  [['truncate','dtrunc','trunc'], 'DECIMAL', ['DECIMAL'], 'impala::DecimalFunctions::Truncate'],
  [['truncate','dtrunc','trunc'], 'DECIMAL', ['DECIMAL', 'TINYINT'],
      'impala::DecimalFunctions::TruncateTo'],
  [['truncate','dtrunc','trunc'], 'DECIMAL', ['DECIMAL', 'SMALLINT'],
      'impala::DecimalFunctions::TruncateTo'],
  [['truncate','dtrunc','trunc'], 'DECIMAL', ['DECIMAL', 'INT'],
      'impala::DecimalFunctions::TruncateTo'],
  [['truncate','dtrunc','trunc'], 'DECIMAL', ['DECIMAL', 'BIGINT'],
      'impala::DecimalFunctions::TruncateTo'],
  # String builtin functions
  [['substr', 'substring'], 'STRING', ['STRING', 'BIGINT'],
   'impala::StringFunctions::Substring'],
  [['substr', 'substring'], 'STRING', ['STRING', 'BIGINT', 'BIGINT'],
   'impala::StringFunctions::Substring'],
  [['utf8_substr', 'utf8_substring'], 'STRING', ['STRING', 'BIGINT'],
   'impala::StringFunctions::Utf8Substring'],
  [['utf8_substr', 'utf8_substring'], 'STRING', ['STRING', 'BIGINT', 'BIGINT'],
   'impala::StringFunctions::Utf8Substring'],
  [['split_part'], 'STRING', ['STRING', 'STRING', 'BIGINT'],
   'impala::StringFunctions::SplitPart'],
  [['base64encode'], 'STRING', ['STRING'], 'impala::StringFunctions::Base64Encode'],
  [['base64decode'], 'STRING', ['STRING'], 'impala::StringFunctions::Base64Decode'],
  [['left', 'strleft'], 'STRING', ['STRING', 'BIGINT'], 'impala::StringFunctions::Left'],
  [['right', 'strright'], 'STRING', ['STRING', 'BIGINT'], 'impala::StringFunctions::Right'],
  [['space'], 'STRING', ['BIGINT'], 'impala::StringFunctions::Space'],
  [['repeat'], 'STRING', ['STRING', 'BIGINT'], 'impala::StringFunctions::Repeat'],
  [['lpad'], 'STRING', ['STRING', 'BIGINT', 'STRING'], 'impala::StringFunctions::Lpad'],
  [['rpad'], 'STRING', ['STRING', 'BIGINT', 'STRING'], 'impala::StringFunctions::Rpad'],
  [['ai_generate_text'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING', 'STRING', 'STRING'],
   'impala::AiFunctions::AiGenerateText'],
  [['ai_generate_text_default'], 'STRING', ['STRING'],
   'impala::AiFunctions::AiGenerateTextDefault'],
  [['bytes'], 'INT', ['STRING'], 'impala::StringFunctions::Bytes'],
  [['aes_encrypt'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING'], 'impala::StringFunctions::AesEncrypt',
   '_ZN6impala15StringFunctions10AesPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions8AesCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['aes_decrypt'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING'], 'impala::StringFunctions::AesDecrypt',
   '_ZN6impala15StringFunctions10AesPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions8AesCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['length'], 'INT', ['STRING'], 'impala::StringFunctions::Length'],
  [['length'], 'INT', ['BINARY'], 'impala::StringFunctions::Bytes'],
  [['length'], 'INT', ['CHAR'], 'impala::StringFunctions::CharLength'],
  [['char_length'], 'INT', ['STRING'], 'impala::StringFunctions::Length'],
  [['character_length'], 'INT', ['STRING'], 'impala::StringFunctions::Length'],
  [['utf8_length'], 'INT', ['STRING'], 'impala::StringFunctions::Utf8Length'],
  [['lower', 'lcase'], 'STRING', ['STRING'], 'impala::StringFunctions::Lower'],
  [['lower_ascii', 'lcase_ascii'], 'STRING', ['STRING'], 'impala::StringFunctions::LowerAscii'],
  [['lower_utf8', 'lcase_utf8'], 'STRING', ['STRING'], 'impala::StringFunctions::LowerUtf8'],
  [['upper', 'ucase'], 'STRING', ['STRING'], 'impala::StringFunctions::Upper'],
  [['upper_ascii', 'ucase_ascii'], 'STRING', ['STRING'], 'impala::StringFunctions::UpperAscii'],
  [['upper_utf8', 'ucase_utf8'], 'STRING', ['STRING'], 'impala::StringFunctions::UpperUtf8'],
  [['initcap'], 'STRING', ['STRING'], 'impala::StringFunctions::InitCap'],
  [['initcap_ascii'], 'STRING', ['STRING'], 'impala::StringFunctions::InitCapAscii'],
  [['initcap_utf8'], 'STRING', ['STRING'], 'impala::StringFunctions::InitCapUtf8'],
  [['replace'], 'STRING', ['STRING', 'STRING', 'STRING'], 'impala::StringFunctions::Replace',
   '_ZN6impala15StringFunctions14ReplacePrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions12ReplaceCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['reverse'], 'STRING', ['STRING'], 'impala::StringFunctions::Reverse'],
  [['utf8_reverse'], 'STRING', ['STRING'], 'impala::StringFunctions::Utf8Reverse'],
  [['translate'], 'STRING', ['STRING', 'STRING', 'STRING'],
   'impala::StringFunctions::Translate'],
  [['trim'], 'STRING', ['STRING'], 'impala::StringFunctions::Trim',
   '_ZN6impala15StringFunctions11TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['ltrim'], 'STRING', ['STRING'], 'impala::StringFunctions::Ltrim',
   '_ZN6impala15StringFunctions11TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['rtrim'], 'STRING', ['STRING'], 'impala::StringFunctions::Rtrim',
   '_ZN6impala15StringFunctions11TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['ltrim'], 'STRING', ['STRING', 'STRING'], 'impala::StringFunctions::LTrimString',
   '_ZN6impala15StringFunctions11TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['rtrim'], 'STRING', ['STRING', 'STRING'], 'impala::StringFunctions::RTrimString',
   '_ZN6impala15StringFunctions11TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['utf8_trim'], 'STRING', ['STRING'], 'impala::StringFunctions::Trim',
   '_ZN6impala15StringFunctions15Utf8TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['utf8_ltrim'], 'STRING', ['STRING'], 'impala::StringFunctions::Ltrim',
   '_ZN6impala15StringFunctions15Utf8TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['utf8_rtrim'], 'STRING', ['STRING'], 'impala::StringFunctions::Rtrim',
   '_ZN6impala15StringFunctions15Utf8TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['utf8_ltrim'], 'STRING', ['STRING', 'STRING'], 'impala::StringFunctions::LTrimString',
   '_ZN6impala15StringFunctions15Utf8TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['utf8_rtrim'], 'STRING', ['STRING', 'STRING'], 'impala::StringFunctions::RTrimString',
   '_ZN6impala15StringFunctions15Utf8TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['ascii'], 'INT', ['STRING'], 'impala::StringFunctions::Ascii'],
  [['instr'], 'INT', ['STRING', 'STRING'], 'impala::StringFunctions::Instr'],
  [['instr'], 'INT', ['STRING', 'STRING', 'BIGINT'], 'impala::StringFunctions::Instr'],
  [['instr'], 'INT', ['STRING', 'STRING', 'BIGINT', 'BIGINT'],
   'impala::StringFunctions::Instr'],
  [['locate'], 'INT', ['STRING', 'STRING'], 'impala::StringFunctions::Locate'],
  [['locate'], 'INT', ['STRING', 'STRING', 'BIGINT'],
   'impala::StringFunctions::LocatePos'],
  [['regexp_escape'], 'STRING', ['STRING'], 'impala::StringFunctions::RegexpEscape'],
  [['regexp_extract'], 'STRING', ['STRING', 'STRING', 'BIGINT'],
   'impala::StringFunctions::RegexpExtract',
   '_ZN6impala15StringFunctions13RegexpPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions11RegexpCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['regexp_replace'], 'STRING', ['STRING', 'STRING', 'STRING'],
   'impala::StringFunctions::RegexpReplace',
   '_ZN6impala15StringFunctions13RegexpPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions11RegexpCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['regexp_like'], 'BOOLEAN', ['STRING', 'STRING'],
   'impala::LikePredicate::Regex',
   '_ZN6impala13LikePredicate12RegexPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala13LikePredicate10RegexCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['regexp_like'], 'BOOLEAN', ['STRING', 'STRING', 'STRING'],
   'impala::LikePredicate::RegexpLike',
   '_ZN6impala13LikePredicate17RegexpLikePrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala13LikePredicate10RegexCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['regexp_match_count'], 'INT', ['STRING', 'STRING'],
   'impala::StringFunctions::RegexpMatchCount2Args',
   '_ZN6impala15StringFunctions23RegexpMatchCountPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions11RegexpCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['regexp_match_count'], 'INT', ['STRING', 'STRING', 'INT', 'STRING'],
   'impala::StringFunctions::RegexpMatchCount4Args',
   '_ZN6impala15StringFunctions23RegexpMatchCountPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions11RegexpCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['concat'], 'STRING', ['STRING', '...'], 'impala::StringFunctions::Concat'],
  [['concat'], 'BINARY', ['BINARY', '...'], 'impala::StringFunctions::Concat'],
  [['concat_ws'], 'STRING', ['STRING', 'STRING', '...'],
   'impala::StringFunctions::ConcatWs'],
  [['find_in_set'], 'INT', ['STRING', 'STRING'], 'impala::StringFunctions::FindInSet'],
  [['parse_url'], 'STRING', ['STRING', 'STRING'], 'impala::StringFunctions::ParseUrl',
   '_ZN6impala15StringFunctions15ParseUrlPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions13ParseUrlCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['parse_url'], 'STRING', ['STRING', 'STRING', 'STRING'], 'impala::StringFunctions::ParseUrlKey',
   '_ZN6impala15StringFunctions15ParseUrlPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions13ParseUrlCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
   [['prettyprint_bytes'], 'STRING', ['BIGINT'], 'impala::StringFunctions::PrettyPrintMemory'],
   [['prettyprint_bytes'], 'STRING', ['INT'], 'impala::StringFunctions::PrettyPrintMemory'],
   [['prettyprint_bytes'], 'STRING', ['SMALLINT'], 'impala::StringFunctions::PrettyPrintMemory'],
   [['prettyprint_bytes'], 'STRING', ['TINYINT'], 'impala::StringFunctions::PrettyPrintMemory'],
# Netezza compatibility char functions
  [['chr'], 'STRING', ['INT'], 'impala::StringFunctions::Chr'],
  [['btrim'], 'STRING', ['STRING'], 'impala::StringFunctions::Trim',
   '_ZN6impala15StringFunctions11TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['btrim'], 'STRING', ['STRING', 'STRING'], 'impala::StringFunctions::BTrimString',
   '_ZN6impala15StringFunctions11TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['utf8_btrim'], 'STRING', ['STRING'], 'impala::StringFunctions::Trim',
   '_ZN6impala15StringFunctions15Utf8TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['utf8_btrim'], 'STRING', ['STRING', 'STRING'], 'impala::StringFunctions::BTrimString',
   '_ZN6impala15StringFunctions15Utf8TrimPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala15StringFunctions9TrimCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['get_json_object'], 'STRING', ['STRING', 'STRING'],
   'impala::StringFunctions::GetJsonObject'],
  [['levenshtein', 'le_dst'], 'INT', ['STRING', 'STRING'],
   '_ZN6impala15StringFunctions11LevenshteinEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['damerau_levenshtein', 'dle_dst'], 'INT', ['STRING', 'STRING'],
   '_ZN6impala15StringFunctions18DamerauLevenshteinEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['jaro_distance', 'jaro_dst'], 'DOUBLE', ['STRING', 'STRING'],
   '_ZN6impala15StringFunctions12JaroDistanceEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['jaro_similarity', 'jaro_sim'], 'DOUBLE', ['STRING', 'STRING'],
   '_ZN6impala15StringFunctions14JaroSimilarityEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['jaro_winkler_distance', 'jw_dst'], 'DOUBLE', ['STRING', 'STRING'],
   '_ZN6impala15StringFunctions19JaroWinklerDistanceEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['jaro_winkler_distance', 'jw_dst'], 'DOUBLE', ['STRING', 'STRING', 'DOUBLE'],
   '_ZN6impala15StringFunctions19JaroWinklerDistanceEPN10impala_udf15FunctionContextERKNS1_9StringValES6_RKNS1_9DoubleValE'],
  [['jaro_winkler_distance', 'jw_dst'], 'DOUBLE', ['STRING', 'STRING', 'DOUBLE', 'DOUBLE'],
   '_ZN6impala15StringFunctions19JaroWinklerDistanceEPN10impala_udf15FunctionContextERKNS1_9StringValES6_RKNS1_9DoubleValES9_'],
  [['jaro_winkler_similarity', 'jw_sim'], 'DOUBLE', ['STRING', 'STRING'],
   '_ZN6impala15StringFunctions21JaroWinklerSimilarityEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['jaro_winkler_similarity', 'jw_sim'], 'DOUBLE', ['STRING', 'STRING', 'DOUBLE'],
   '_ZN6impala15StringFunctions21JaroWinklerSimilarityEPN10impala_udf15FunctionContextERKNS1_9StringValES6_RKNS1_9DoubleValE'],
  [['jaro_winkler_similarity', 'jw_sim'], 'DOUBLE', ['STRING', 'STRING', 'DOUBLE', 'DOUBLE'],
   '_ZN6impala15StringFunctions21JaroWinklerSimilarityEPN10impala_udf15FunctionContextERKNS1_9StringValES6_RKNS1_9DoubleValES9_'],
  # Conditional Functions
  # Some of these have empty symbols because the BE special-cases them based on the
  # function name
  [['if'], 'BOOLEAN', ['BOOLEAN', 'BOOLEAN', 'BOOLEAN'], ''],
  [['if'], 'TINYINT', ['BOOLEAN', 'TINYINT', 'TINYINT'], ''],
  [['if'], 'SMALLINT', ['BOOLEAN', 'SMALLINT', 'SMALLINT'], ''],
  [['if'], 'INT', ['BOOLEAN', 'INT', 'INT'], ''],
  [['if'], 'BIGINT', ['BOOLEAN', 'BIGINT', 'BIGINT'], ''],
  [['if'], 'FLOAT', ['BOOLEAN', 'FLOAT', 'FLOAT'], ''],
  [['if'], 'DOUBLE', ['BOOLEAN', 'DOUBLE', 'DOUBLE'], ''],
  [['if'], 'STRING', ['BOOLEAN', 'STRING', 'STRING'], ''],
  [['if'], 'BINARY', ['BOOLEAN', 'BINARY', 'BINARY'], ''],
  [['if'], 'TIMESTAMP', ['BOOLEAN', 'TIMESTAMP', 'TIMESTAMP'], ''],
  [['if'], 'DECIMAL', ['BOOLEAN', 'DECIMAL', 'DECIMAL'], ''],
  [['if'], 'DATE', ['BOOLEAN', 'DATE', 'DATE'], ''],
  [['zeroifnull'], 'TINYINT', ['TINYINT'], 'impala::ConditionalFunctions::ZeroIfNull'],
  [['zeroifnull'], 'SMALLINT', ['SMALLINT'], 'impala::ConditionalFunctions::ZeroIfNull'],
  [['zeroifnull'], 'INT', ['INT'], 'impala::ConditionalFunctions::ZeroIfNull'],
  [['zeroifnull'], 'BIGINT', ['BIGINT'], 'impala::ConditionalFunctions::ZeroIfNull'],
  [['zeroifnull'], 'FLOAT', ['FLOAT'], 'impala::ConditionalFunctions::ZeroIfNull'],
  [['zeroifnull'], 'DOUBLE', ['DOUBLE'], 'impala::ConditionalFunctions::ZeroIfNull'],
  [['zeroifnull'], 'DECIMAL', ['DECIMAL'], 'impala::ConditionalFunctions::ZeroIfNull'],
  [['nullifzero'], 'TINYINT', ['TINYINT'], 'impala::ConditionalFunctions::NullIfZero'],
  [['nullifzero'], 'SMALLINT', ['SMALLINT'], 'impala::ConditionalFunctions::NullIfZero'],
  [['nullifzero'], 'INT', ['INT'], 'impala::ConditionalFunctions::NullIfZero'],
  [['nullifzero'], 'BIGINT', ['BIGINT'], 'impala::ConditionalFunctions::NullIfZero'],
  [['nullifzero'], 'FLOAT', ['FLOAT'], 'impala::ConditionalFunctions::NullIfZero'],
  [['nullifzero'], 'DOUBLE', ['DOUBLE'], 'impala::ConditionalFunctions::NullIfZero'],
  [['nullifzero'], 'DECIMAL', ['DECIMAL'], 'impala::ConditionalFunctions::NullIfZero'],
  [['isnull', 'ifnull', 'nvl'], 'BOOLEAN', ['BOOLEAN', 'BOOLEAN'], ''],
  [['isnull', 'ifnull', 'nvl'], 'TINYINT', ['TINYINT', 'TINYINT'], ''],
  [['isnull', 'ifnull', 'nvl'], 'SMALLINT', ['SMALLINT', 'SMALLINT'], ''],
  [['isnull', 'ifnull', 'nvl'], 'INT', ['INT', 'INT'], ''],
  [['isnull', 'ifnull', 'nvl'], 'BIGINT', ['BIGINT', 'BIGINT'], ''],
  [['isnull', 'ifnull', 'nvl'], 'FLOAT', ['FLOAT', 'FLOAT'], ''],
  [['isnull', 'ifnull', 'nvl'], 'DOUBLE', ['DOUBLE', 'DOUBLE'], ''],
  [['isnull', 'ifnull', 'nvl'], 'STRING', ['STRING', 'STRING'], ''],
  [['isnull', 'ifnull', 'nvl'], 'BINARY', ['BINARY', 'BINARY'], ''],
  [['isnull', 'ifnull', 'nvl'], 'TIMESTAMP', ['TIMESTAMP', 'TIMESTAMP'], ''],
  [['isnull', 'ifnull', 'nvl'], 'DECIMAL', ['DECIMAL', 'DECIMAL'], ''],
  [['isnull', 'ifnull', 'nvl'], 'DATE', ['DATE', 'DATE'], ''],
  [['coalesce'], 'BOOLEAN', ['BOOLEAN', '...'], ''],
  [['coalesce'], 'TINYINT', ['TINYINT', '...'], ''],
  [['coalesce'], 'SMALLINT', ['SMALLINT', '...'], ''],
  [['coalesce'], 'INT', ['INT', '...'], ''],
  [['coalesce'], 'BIGINT', ['BIGINT', '...'], ''],
  [['coalesce'], 'FLOAT', ['FLOAT', '...'], ''],
  [['coalesce'], 'DOUBLE', ['DOUBLE', '...'], ''],
  [['coalesce'], 'STRING', ['STRING', '...'], ''],
  [['coalesce'], 'BINARY', ['BINARY', '...'], ''],
  [['coalesce'], 'TIMESTAMP', ['TIMESTAMP', '...'], ''],
  [['coalesce'], 'DECIMAL', ['DECIMAL', '...'], ''],
  [['coalesce'], 'DATE', ['DATE', '...'], ''],
  [['istrue'], 'BOOLEAN', ['BOOLEAN'], 'impala::ConditionalFunctions::IsTrue'],
  [['isnottrue'], 'BOOLEAN', ['BOOLEAN'], 'impala::ConditionalFunctions::IsNotTrue'],
  [['isfalse'], 'BOOLEAN', ['BOOLEAN'], 'impala::ConditionalFunctions::IsFalse'],
  [['isnotfalse'], 'BOOLEAN', ['BOOLEAN'], 'impala::ConditionalFunctions::IsNotFalse'],
  # Utility functions
  [['uuid'], 'STRING', [],
   '_ZN6impala16UtilityFunctions4UuidEPN10impala_udf15FunctionContextE',
   '_ZN6impala16UtilityFunctions11UuidPrepareEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE',
   '_ZN6impala16UtilityFunctions9UuidCloseEPN10impala_udf15FunctionContextENS2_18FunctionStateScopeE'],
  [['current_database', 'current_catalog'], 'STRING', [], 'impala::UtilityFunctions::CurrentDatabase'],
  [['current_session', 'current_sid'], 'STRING', [], 'impala::UtilityFunctions::CurrentSession'],
  [['user', 'session_user'], 'STRING', [], 'impala::UtilityFunctions::User'],
  [['effective_user', 'current_user', 'logged_in_user'], 'STRING', [],
      'impala::UtilityFunctions::EffectiveUser'],
  [['sleep'], 'BOOLEAN', ['INT'], 'impala::UtilityFunctions::Sleep'],
  [['pid'], 'INT', [], 'impala::UtilityFunctions::Pid'],
  [['version'], 'STRING', [], 'impala::UtilityFunctions::Version'],
  [['coordinator'], 'STRING', [], 'impala::UtilityFunctions::Coordinator'],
  [['typeOf'], 'STRING', ['BOOLEAN'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf10BooleanValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['TINYINT'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf10TinyIntValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['SMALLINT'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf11SmallIntValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['INT'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf6IntValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['BIGINT'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf9BigIntValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['FLOAT'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf8FloatValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['DOUBLE'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf9DoubleValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['CHAR'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf9StringValEEES3_PNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['VARCHAR'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf9StringValEEES3_PNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['STRING'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf9StringValEEES3_PNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['TIMESTAMP'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf12TimestampValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['DECIMAL'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf10DecimalValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['DATE'], '_ZN6impala16UtilityFunctions6TypeOfIN10impala_udf7DateValEEENS2_9StringValEPNS2_15FunctionContextERKT_'],
  [['typeOf'], 'STRING', ['BINARY'], '_ZN6impala16UtilityFunctions12TypeOfBinaryEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['fnv_hash'], 'BIGINT', ['TINYINT'],
   '_ZN6impala16UtilityFunctions7FnvHashIN10impala_udf10TinyIntValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['fnv_hash'], 'BIGINT', ['SMALLINT'],
   '_ZN6impala16UtilityFunctions7FnvHashIN10impala_udf11SmallIntValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['fnv_hash'], 'BIGINT', ['INT'],
   '_ZN6impala16UtilityFunctions7FnvHashIN10impala_udf6IntValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['fnv_hash'], 'BIGINT', ['BIGINT'],
   '_ZN6impala16UtilityFunctions7FnvHashIN10impala_udf9BigIntValEEES3_PNS2_15FunctionContextERKT_'],
  [['fnv_hash'], 'BIGINT', ['FLOAT'],
   '_ZN6impala16UtilityFunctions7FnvHashIN10impala_udf8FloatValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['fnv_hash'], 'BIGINT', ['DOUBLE'],
   '_ZN6impala16UtilityFunctions7FnvHashIN10impala_udf9DoubleValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['fnv_hash'], 'BIGINT', ['DATE'],
   '_ZN6impala16UtilityFunctions7FnvHashIN10impala_udf7DateValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['fnv_hash'], 'BIGINT', ['STRING'],
   '_ZN6impala16UtilityFunctions13FnvHashStringEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['fnv_hash'], 'BIGINT', ['TIMESTAMP'],
   '_ZN6impala16UtilityFunctions16FnvHashTimestampEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['fnv_hash'], 'BIGINT', ['DECIMAL'],
   '_ZN6impala16UtilityFunctions14FnvHashDecimalEPN10impala_udf15FunctionContextERKNS1_10DecimalValE'],
  [['murmur_hash'], 'BIGINT', ['TINYINT'],
   '_ZN6impala16UtilityFunctions10MurmurHashIN10impala_udf10TinyIntValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['murmur_hash'], 'BIGINT', ['SMALLINT'],
   '_ZN6impala16UtilityFunctions10MurmurHashIN10impala_udf11SmallIntValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['murmur_hash'], 'BIGINT', ['INT'],
   '_ZN6impala16UtilityFunctions10MurmurHashIN10impala_udf6IntValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['murmur_hash'], 'BIGINT', ['BIGINT'],
   '_ZN6impala16UtilityFunctions10MurmurHashIN10impala_udf9BigIntValEEES3_PNS2_15FunctionContextERKT_'],
  [['murmur_hash'], 'BIGINT', ['FLOAT'],
   '_ZN6impala16UtilityFunctions10MurmurHashIN10impala_udf8FloatValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['murmur_hash'], 'BIGINT', ['DOUBLE'],
   '_ZN6impala16UtilityFunctions10MurmurHashIN10impala_udf9DoubleValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['murmur_hash'], 'BIGINT', ['DATE'],
   '_ZN6impala16UtilityFunctions10MurmurHashIN10impala_udf7DateValEEENS2_9BigIntValEPNS2_15FunctionContextERKT_'],
  [['murmur_hash'], 'BIGINT', ['STRING'],
   '_ZN6impala16UtilityFunctions16MurmurHashStringEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['murmur_hash'], 'BIGINT', ['BINARY'],
   '_ZN6impala16UtilityFunctions16MurmurHashStringEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['murmur_hash'], 'BIGINT', ['TIMESTAMP'],
   '_ZN6impala16UtilityFunctions19MurmurHashTimestampEPN10impala_udf15FunctionContextERKNS1_12TimestampValE'],
  [['murmur_hash'], 'BIGINT', ['DECIMAL'],
   '_ZN6impala16UtilityFunctions17MurmurHashDecimalEPN10impala_udf15FunctionContextERKNS1_10DecimalValE'],
  [['sha1'], 'STRING', ['STRING'],
   '_ZN6impala16UtilityFunctions4Sha1EPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['sha2'], 'STRING', ['STRING', 'INT'],
   '_ZN6impala16UtilityFunctions4Sha2EPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_6IntValE'],
  [['md5'], 'STRING', ['STRING'],
   '_ZN6impala16UtilityFunctions3Md5EPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  # (Non)NullValue functions
  [['nullvalue'], 'BOOLEAN', ['BOOLEAN'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf10BooleanValEEES3_PNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['TINYINT'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf10TinyIntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['SMALLINT'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf11SmallIntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['INT'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf6IntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['BIGINT'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf9BigIntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['FLOAT'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf8FloatValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['DOUBLE'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf9DoubleValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['STRING'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf9StringValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['TIMESTAMP'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf12TimestampValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nullvalue'], 'BOOLEAN', ['DECIMAL'], '_ZN6impala15IsNullPredicate6IsNullIN10impala_udf10DecimalValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['BOOLEAN'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf10BooleanValEEES3_PNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['TINYINT'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf10TinyIntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['SMALLINT'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf11SmallIntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['INT'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf6IntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['BIGINT'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf9BigIntValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['FLOAT'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf8FloatValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['DOUBLE'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf9DoubleValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['STRING'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf9StringValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['TIMESTAMP'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf12TimestampValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  [['nonnullvalue'], 'BOOLEAN', ['DECIMAL'], '_ZN6impala15IsNullPredicate9IsNotNullIN10impala_udf10DecimalValEEENS2_10BooleanValEPNS2_15FunctionContextERKT_'],
  # Bit and Byte functions
  # For functions corresponding to builtin operators, we can reuse the implementations
  [['bitand'], 'TINYINT', ['TINYINT', 'TINYINT'], 'impala::Operators::Bitand_TinyIntVal_TinyIntVal'],
  [['bitand'], 'SMALLINT', ['SMALLINT', 'SMALLINT'], 'impala::Operators::Bitand_SmallIntVal_SmallIntVal'],
  [['bitand'], 'INT', ['INT', 'INT'], 'impala::Operators::Bitand_IntVal_IntVal'],
  [['bitand'], 'BIGINT', ['BIGINT', 'BIGINT'], 'impala::Operators::Bitand_BigIntVal_BigIntVal'],
  [['bitor'], 'TINYINT', ['TINYINT', 'TINYINT'], 'impala::Operators::Bitor_TinyIntVal_TinyIntVal'],
  [['bitor'], 'SMALLINT', ['SMALLINT', 'SMALLINT'], 'impala::Operators::Bitor_SmallIntVal_SmallIntVal'],
  [['bitor'], 'INT', ['INT', 'INT'], 'impala::Operators::Bitor_IntVal_IntVal'],
  [['bitor'], 'BIGINT', ['BIGINT', 'BIGINT'], 'impala::Operators::Bitor_BigIntVal_BigIntVal'],
  [['bitxor'], 'TINYINT', ['TINYINT', 'TINYINT'], 'impala::Operators::Bitxor_TinyIntVal_TinyIntVal'],
  [['bitxor'], 'SMALLINT', ['SMALLINT', 'SMALLINT'], 'impala::Operators::Bitxor_SmallIntVal_SmallIntVal'],
  [['bitxor'], 'INT', ['INT', 'INT'], 'impala::Operators::Bitxor_IntVal_IntVal'],
  [['bitxor'], 'BIGINT', ['BIGINT', 'BIGINT'], 'impala::Operators::Bitxor_BigIntVal_BigIntVal'],
  [['bitnot'], 'TINYINT', ['TINYINT'], 'impala::Operators::Bitnot_TinyIntVal'],
  [['bitnot'], 'SMALLINT', ['SMALLINT'], 'impala::Operators::Bitnot_SmallIntVal'],
  [['bitnot'], 'INT', ['INT'], 'impala::Operators::Bitnot_IntVal'],
  [['bitnot'], 'BIGINT', ['BIGINT'], 'impala::Operators::Bitnot_BigIntVal'],
  [['countset'], 'INT', ['TINYINT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf10TinyIntValEEENS2_6IntValEPNS2_15FunctionContextERKT_'],
  [['countset'], 'INT', ['SMALLINT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf11SmallIntValEEENS2_6IntValEPNS2_15FunctionContextERKT_'],
  [['countset'], 'INT', ['INT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf6IntValEEES3_PNS2_15FunctionContextERKT_'],
  [['countset'], 'INT', ['BIGINT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf9BigIntValEEENS2_6IntValEPNS2_15FunctionContextERKT_'],
  [['countset'], 'INT', ['TINYINT', 'INT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf10TinyIntValEEENS2_6IntValEPNS2_15FunctionContextERKT_RKS4_'],
  [['countset'], 'INT', ['SMALLINT', 'INT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf11SmallIntValEEENS2_6IntValEPNS2_15FunctionContextERKT_RKS4_'],
  [['countset'], 'INT', ['INT', 'INT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf6IntValEEES3_PNS2_15FunctionContextERKT_RKS3_'],
  [['countset'], 'INT', ['BIGINT', 'INT'], '_ZN6impala16BitByteFunctions8CountSetIN10impala_udf9BigIntValEEENS2_6IntValEPNS2_15FunctionContextERKT_RKS4_'],
  [['getbit'], 'TINYINT', ['TINYINT', 'INT'], '_ZN6impala16BitByteFunctions6GetBitIN10impala_udf10TinyIntValEEES3_PNS2_15FunctionContextERKT_RKNS2_6IntValE'],
  [['getbit'], 'TINYINT', ['SMALLINT', 'INT'], '_ZN6impala16BitByteFunctions6GetBitIN10impala_udf11SmallIntValEEENS2_10TinyIntValEPNS2_15FunctionContextERKT_RKNS2_6IntValE'],
  [['getbit'], 'TINYINT', ['INT', 'INT'], '_ZN6impala16BitByteFunctions6GetBitIN10impala_udf6IntValEEENS2_10TinyIntValEPNS2_15FunctionContextERKT_RKS3_'],
  [['getbit'], 'TINYINT', ['BIGINT', 'INT'], '_ZN6impala16BitByteFunctions6GetBitIN10impala_udf9BigIntValEEENS2_10TinyIntValEPNS2_15FunctionContextERKT_RKNS2_6IntValE'],
  [['rotateleft'], 'TINYINT', ['TINYINT', 'INT'], 'impala::BitByteFunctions::RotateLeft'],
  [['rotateleft'], 'SMALLINT', ['SMALLINT', 'INT'], 'impala::BitByteFunctions::RotateLeft'],
  [['rotateleft'], 'INT', ['INT', 'INT'], 'impala::BitByteFunctions::RotateLeft'],
  [['rotateleft'], 'BIGINT', ['BIGINT', 'INT'], 'impala::BitByteFunctions::RotateLeft'],
  [['rotateright'], 'TINYINT', ['TINYINT', 'INT'], 'impala::BitByteFunctions::RotateRight'],
  [['rotateright'], 'SMALLINT', ['SMALLINT', 'INT'], 'impala::BitByteFunctions::RotateRight'],
  [['rotateright'], 'INT', ['INT', 'INT'], 'impala::BitByteFunctions::RotateRight'],
  [['rotateright'], 'BIGINT', ['BIGINT', 'INT'], 'impala::BitByteFunctions::RotateRight'],
  [['setbit'], 'TINYINT', ['TINYINT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf10TinyIntValEEET_PNS2_15FunctionContextERKS4_RKNS2_6IntValE'],
  [['setbit'], 'SMALLINT', ['SMALLINT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf11SmallIntValEEET_PNS2_15FunctionContextERKS4_RKNS2_6IntValE'],
  [['setbit'], 'INT', ['INT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf6IntValEEET_PNS2_15FunctionContextERKS4_RKS3_'],
  [['setbit'], 'BIGINT', ['BIGINT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf9BigIntValEEET_PNS2_15FunctionContextERKS4_RKNS2_6IntValE'],
  [['setbit'], 'TINYINT', ['TINYINT', 'INT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf10TinyIntValEEET_PNS2_15FunctionContextERKS4_RKNS2_6IntValESB_'],
  [['setbit'], 'SMALLINT', ['SMALLINT', 'INT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf11SmallIntValEEET_PNS2_15FunctionContextERKS4_RKNS2_6IntValESB_'],
  [['setbit'], 'INT', ['INT', 'INT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf6IntValEEET_PNS2_15FunctionContextERKS4_RKS3_SA_'],
  [['setbit'], 'BIGINT', ['BIGINT', 'INT', 'INT'], '_ZN6impala16BitByteFunctions6SetBitIN10impala_udf9BigIntValEEET_PNS2_15FunctionContextERKS4_RKNS2_6IntValESB_'],
  [['shiftleft'], 'TINYINT', ['TINYINT', 'INT'], 'impala::BitByteFunctions::ShiftLeft'],
  [['shiftleft'], 'SMALLINT', ['SMALLINT', 'INT'], 'impala::BitByteFunctions::ShiftLeft'],
  [['shiftleft'], 'INT', ['INT', 'INT'], 'impala::BitByteFunctions::ShiftLeft'],
  [['shiftleft'], 'BIGINT', ['BIGINT', 'INT'], 'impala::BitByteFunctions::ShiftLeft'],
  [['shiftright'], 'TINYINT', ['TINYINT', 'INT'], 'impala::BitByteFunctions::ShiftRight'],
  [['shiftright'], 'SMALLINT', ['SMALLINT', 'INT'], 'impala::BitByteFunctions::ShiftRight'],
  [['shiftright'], 'INT', ['INT', 'INT'], 'impala::BitByteFunctions::ShiftRight'],
  [['shiftright'], 'BIGINT', ['BIGINT', 'INT'], 'impala::BitByteFunctions::ShiftRight'],
  # Overloads of mask_show_first_n
  [['mask_show_first_n'], 'STRING', ['STRING'], 'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'STRING', ['STRING', 'INT'], 'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'BIGINT', ['BIGINT'], 'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'BIGINT', ['BIGINT', 'INT'], 'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'BIGINT', ['BIGINT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskShowFirstN'],
  [['mask_show_first_n'], 'DATE', ['DATE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  # mask_show_first_n always return null for other types
  [['mask_show_first_n'], 'BOOLEAN', ['BOOLEAN', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_show_first_n'], 'DOUBLE', ['DOUBLE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_show_first_n'], 'TIMESTAMP', ['TIMESTAMP', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  # Overloads of mask_show_last_n
  [['mask_show_last_n'], 'STRING', ['STRING'], 'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'STRING', ['STRING', 'INT'], 'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING'],
      'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'BIGINT', ['BIGINT'], 'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'BIGINT', ['BIGINT', 'INT'], 'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowLastN'],
  [['mask_show_last_n'], 'BIGINT', ['BIGINT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskShowLastN'],
  # Reuse the implementation of mask_show_first_n on date type.
  # TODO: implement this after HIVE-24641 is resolved.
  [['mask_show_last_n'], 'DATE', ['DATE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  # mask_show_last_n always return null for other types
  [['mask_show_last_n'], 'BOOLEAN', ['BOOLEAN', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_show_last_n'], 'DOUBLE', ['DOUBLE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_show_last_n'], 'TIMESTAMP', ['TIMESTAMP', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  # Overloads of mask_first_n
  [['mask_first_n'], 'STRING', ['STRING'], 'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'STRING', ['STRING', 'INT'], 'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING'],
      'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'BIGINT', ['BIGINT'], 'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'BIGINT', ['BIGINT', 'INT'], 'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskFirstN'],
  [['mask_first_n'], 'BIGINT', ['BIGINT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskFirstN'],
  # Reuse the implementation of mask_show_first_n on date type.
  # TODO: implement this after HIVE-24641 is resolved.
  [['mask_first_n'], 'DATE', ['DATE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  # mask_first_n always return null for other types
  [['mask_first_n'], 'BOOLEAN', ['BOOLEAN', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_first_n'], 'DOUBLE', ['DOUBLE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_first_n'], 'TIMESTAMP', ['TIMESTAMP', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  # Overloads of mask_last_n
  [['mask_last_n'], 'STRING', ['STRING'], 'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'STRING', ['STRING', 'INT'], 'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING'],
      'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'STRING', ['STRING', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'BIGINT', ['BIGINT'], 'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'BIGINT', ['BIGINT', 'INT'], 'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'BIGINT', ['BIGINT', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskLastN'],
  [['mask_last_n'], 'BIGINT', ['BIGINT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::MaskLastN'],
  # Reuse the implementation of mask_show_first_n on date type.
  # TODO: implement this after HIVE-24641 is resolved.
  [['mask_last_n'], 'DATE', ['DATE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskShowFirstN'],
  # mask_last_n always return null for other types
  [['mask_last_n'], 'BOOLEAN', ['BOOLEAN', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_last_n'], 'DOUBLE', ['DOUBLE', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  [['mask_last_n'], 'TIMESTAMP', ['TIMESTAMP', 'INT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::MaskNull'],
  # Overloads of mask
  [['mask'], 'STRING', ['STRING'], 'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING', 'STRING'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'STRING'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING', 'STRING', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'STRING', 'STRING', 'STRING', 'INT', 'STRING', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'STRING', ['STRING', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'BIGINT', ['BIGINT'], 'impala::MaskFunctions::Mask'],
  [['mask'], 'BIGINT', ['BIGINT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'BIGINT', ['BIGINT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING'],
       'impala::MaskFunctions::Mask'],
  [['mask'], 'BIGINT', ['BIGINT', 'INT', 'INT', 'INT', 'INT', 'INT'],
       'impala::MaskFunctions::Mask'],
  [['mask'], 'BIGINT', ['BIGINT', 'STRING', 'STRING', 'STRING', 'STRING', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'BIGINT', ['BIGINT', 'STRING', 'STRING', 'STRING', 'INT', 'STRING', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'BIGINT', ['BIGINT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'DATE', ['DATE'], 'impala::MaskFunctions::Mask'],
  [['mask'], 'DATE', ['DATE', 'STRING', 'STRING', 'STRING', 'STRING', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'DATE', ['DATE', 'STRING', 'STRING', 'STRING', 'STRING', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'DATE', ['DATE', 'STRING', 'STRING', 'STRING', 'INT', 'STRING', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'DATE', ['DATE', 'STRING', 'STRING', 'STRING', 'STRING', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'DATE', ['DATE', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'BOOLEAN', ['BOOLEAN'], 'impala::MaskFunctions::Mask'],
  [['mask'], 'BOOLEAN', ['BOOLEAN', 'STRING', 'STRING', 'STRING', 'INT', 'STRING', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'DOUBLE', ['DOUBLE'], 'impala::MaskFunctions::Mask'],
  [['mask'], 'DOUBLE', ['DOUBLE', 'STRING', 'STRING', 'STRING', 'INT', 'STRING', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  [['mask'], 'TIMESTAMP', ['TIMESTAMP'], 'impala::MaskFunctions::Mask'],
  [['mask'], 'TIMESTAMP', ['TIMESTAMP', 'STRING', 'STRING', 'STRING', 'INT', 'STRING', 'INT', 'INT', 'INT'],
      'impala::MaskFunctions::Mask'],
  # Overloads of mask_hash
  [['mask_hash'], 'STRING', ['STRING'], 'impala::MaskFunctions::MaskHash'],
  [['mask_hash'], 'BIGINT', ['BIGINT'], 'impala::MaskFunctions::MaskHash'],
  [['mask_hash'], 'DOUBLE', ['DOUBLE'], 'impala::MaskFunctions::MaskHash'],
  [['mask_hash'], 'BOOLEAN', ['BOOLEAN'], 'impala::MaskFunctions::MaskHash'],
  [['mask_hash'], 'TIMESTAMP', ['TIMESTAMP'], 'impala::MaskFunctions::MaskHash'],
  [['mask_hash'], 'DATE', ['DATE'], 'impala::MaskFunctions::MaskHash'],
  # Functions to use Apache DataSketches functionality
  [['ds_hll_estimate'], 'BIGINT', ['STRING'],
      '_ZN6impala21DataSketchesFunctions13DsHllEstimateEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['ds_hll_estimate_bounds_as_string'], 'STRING', ['STRING'],
      '_ZN6impala21DataSketchesFunctions27DsHllEstimateBoundsAsStringEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['ds_hll_estimate_bounds_as_string'], 'STRING', ['STRING', 'INT'],
      '_ZN6impala21DataSketchesFunctions27DsHllEstimateBoundsAsStringEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_6IntValE'],
  [['ds_hll_union_f'], 'STRING', ['STRING', 'STRING'],
      '_ZN6impala21DataSketchesFunctions11DsHllUnionFEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['ds_hll_stringify'], 'STRING', ['STRING'],
      '_ZN6impala21DataSketchesFunctions14DsHllStringifyEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['ds_cpc_estimate'], 'BIGINT', ['STRING'],
      '_ZN6impala21DataSketchesFunctions13DsCpcEstimateEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['ds_cpc_stringify'], 'STRING', ['STRING'],
     '_ZN6impala21DataSketchesFunctions14DsCpcStringifyEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['ds_cpc_union_f'], 'STRING', ['STRING', 'STRING'],
     '_ZN6impala21DataSketchesFunctions11DsCpcUnionFEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['ds_theta_estimate'], 'BIGINT', ['STRING'],
     '_ZN6impala21DataSketchesFunctions15DsThetaEstimateEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['ds_theta_exclude'], 'STRING', ['STRING', 'STRING'],
     '_ZN6impala21DataSketchesFunctions14DsThetaExcludeEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['ds_theta_union_f'], 'STRING', ['STRING', 'STRING'],
     '_ZN6impala21DataSketchesFunctions13DsThetaUnionFEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['ds_theta_intersect_f'], 'STRING', ['STRING', 'STRING'],
     '_ZN6impala21DataSketchesFunctions17DsThetaIntersectFEPN10impala_udf15FunctionContextERKNS1_9StringValES6_'],
  [['ds_kll_quantile'], 'FLOAT', ['STRING', 'DOUBLE'],
      '_ZN6impala21DataSketchesFunctions13DsKllQuantileEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_9DoubleValE'],
  [['ds_kll_n'], 'BIGINT', ['STRING'],
      '_ZN6impala21DataSketchesFunctions6DsKllNEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
  [['ds_kll_rank'], 'DOUBLE', ['STRING', 'FLOAT'],
      '_ZN6impala21DataSketchesFunctions9DsKllRankEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_8FloatValE'],
  [['ds_kll_quantiles_as_string'], 'STRING', ['STRING', 'DOUBLE', '...'],
      '_ZN6impala21DataSketchesFunctions22DsKllQuantilesAsStringEPN10impala_udf15FunctionContextERKNS1_9StringValEiPKNS1_9DoubleValE'],
  [['ds_kll_pmf_as_string'], 'STRING', ['STRING', 'FLOAT', '...'],
      '_ZN6impala21DataSketchesFunctions16DsKllPMFAsStringEPN10impala_udf15FunctionContextERKNS1_9StringValEiPKNS1_8FloatValE'],
  [['ds_kll_cdf_as_string'], 'STRING', ['STRING', 'FLOAT', '...'],
      '_ZN6impala21DataSketchesFunctions16DsKllCDFAsStringEPN10impala_udf15FunctionContextERKNS1_9StringValEiPKNS1_8FloatValE'],
  [['ds_kll_stringify'], 'STRING', ['STRING'],
      '_ZN6impala21DataSketchesFunctions14DsKllStringifyEPN10impala_udf15FunctionContextERKNS1_9StringValE'],
]
invisible_functions = [
  [['to_utc_timestamp'], 'TIMESTAMP', ['TIMESTAMP', 'STRING', 'BOOLEAN'],
      "impala::TimestampFunctions::ToUtcUnambiguous"],
  [['months_add_interval'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf6IntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_add_interval'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb1EN10impala_udf9BigIntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_sub_interval'], 'TIMESTAMP', ['TIMESTAMP', 'INT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf6IntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_sub_interval'], 'TIMESTAMP', ['TIMESTAMP', 'BIGINT'],
      '_ZN6impala18TimestampFunctions6AddSubILb0EN10impala_udf9BigIntValEN5boost9date_time15months_durationINS4_9gregorian21greg_durations_configEEELb0EEENS2_12TimestampValEPNS2_15FunctionContextERKSA_RKT0_'],
  [['months_add_interval'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb1EN10impala_udf6IntValELb0EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['months_add_interval'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb1EN10impala_udf9BigIntValELb0EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['months_sub_interval'], 'DATE', ['DATE', 'INT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb0EN10impala_udf6IntValELb0EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['months_sub_interval'], 'DATE', ['DATE', 'BIGINT'],
      '_ZN6impala13DateFunctions12AddSubMonthsILb0EN10impala_udf9BigIntValELb0EEENS2_7DateValEPNS2_15FunctionContextERKS4_RKT0_'],
  [['distinctfrom'], 'BOOLEAN', ['BOOLEAN', 'BOOLEAN'], 'impala::Operators::DistinctFrom_BooleanVal_BooleanVal'],
  [['distinctfrom'], 'BOOLEAN', ['TINYINT', 'TINYINT'], 'impala::Operators::DistinctFrom_TinyIntVal_TinyIntVal'],
  [['distinctfrom'], 'BOOLEAN', ['SMALLINT', 'SMALLINT'], 'impala::Operators::DistinctFrom_SmallIntVal_SmallIntVal'],
  [['distinctfrom'], 'BOOLEAN', ['INT', 'INT'], 'impala::Operators::DistinctFrom_IntVal_IntVal'],
  [['distinctfrom'], 'BOOLEAN', ['BIGINT', 'BIGINT'], 'impala::Operators::DistinctFrom_BigIntVal_BigIntVal'],
  [['distinctfrom'], 'BOOLEAN', ['FLOAT', 'FLOAT'], 'impala::Operators::DistinctFrom_FloatVal_FloatVal'],
  [['distinctfrom'], 'BOOLEAN', ['DOUBLE', 'DOUBLE'], 'impala::Operators::DistinctFrom_DoubleVal_DoubleVal'],
  [['distinctfrom'], 'BOOLEAN', ['STRING', 'STRING'], 'impala::Operators::DistinctFrom_StringVal_StringVal'],
  [['distinctfrom'], 'BOOLEAN', ['BINARY', 'BINARY'], 'impala::Operators::DistinctFrom_StringVal_StringVal'],
  [['distinctfrom'], 'BOOLEAN', ['TIMESTAMP', 'TIMESTAMP'], 'impala::Operators::DistinctFrom_TimestampVal_TimestampVal'],
  [['distinctfrom'], 'BOOLEAN', ['CHAR', 'CHAR'], 'impala::Operators::DistinctFrom_Char_Char'],
  [['distinctfrom'], 'BOOLEAN', ['DECIMAL', 'DECIMAL'], 'impala::DecimalOperators::DistinctFrom_DecimalVal_DecimalVal'],
  [['distinctfrom'], 'BOOLEAN', ['DATE', 'DATE'], 'impala::Operators::DistinctFrom_DateVal_DateVal'],
  [['notdistinct'], 'BOOLEAN', ['BOOLEAN', 'BOOLEAN'], 'impala::Operators::NotDistinct_BooleanVal_BooleanVal'],
  [['notdistinct'], 'BOOLEAN', ['TINYINT', 'TINYINT'], 'impala::Operators::NotDistinct_TinyIntVal_TinyIntVal'],
  [['notdistinct'], 'BOOLEAN', ['SMALLINT', 'SMALLINT'], 'impala::Operators::NotDistinct_SmallIntVal_SmallIntVal'],
  [['notdistinct'], 'BOOLEAN', ['INT', 'INT'], 'impala::Operators::NotDistinct_IntVal_IntVal'],
  [['notdistinct'], 'BOOLEAN', ['BIGINT', 'BIGINT'], 'impala::Operators::NotDistinct_BigIntVal_BigIntVal'],
  [['notdistinct'], 'BOOLEAN', ['FLOAT', 'FLOAT'], 'impala::Operators::NotDistinct_FloatVal_FloatVal'],
  [['notdistinct'], 'BOOLEAN', ['DOUBLE', 'DOUBLE'], 'impala::Operators::NotDistinct_DoubleVal_DoubleVal'],
  [['notdistinct'], 'BOOLEAN', ['STRING', 'STRING'], 'impala::Operators::NotDistinct_StringVal_StringVal'],
  [['notdistinct'], 'BOOLEAN', ['BINARY', 'BINARY'], 'impala::Operators::NotDistinct_StringVal_StringVal'],
  [['notdistinct'], 'BOOLEAN', ['TIMESTAMP', 'TIMESTAMP'], 'impala::Operators::NotDistinct_TimestampVal_TimestampVal'],
  [['notdistinct'], 'BOOLEAN', ['CHAR', 'CHAR'], 'impala::Operators::NotDistinct_Char_Char'],
  [['notdistinct'], 'BOOLEAN', ['DECIMAL', 'DECIMAL'], 'impala::DecimalOperators::NotDistinct_DecimalVal_DecimalVal'],
  [['notdistinct'], 'BOOLEAN', ['DATE', 'DATE'], 'impala::Operators::NotDistinct_DateVal_DateVal'],
  # Functions related to Apache Iceberg functionality.
  [['iceberg_truncate_transform'], 'INT', ['INT', 'INT'],
      '_ZN6impala16IcebergFunctions26TruncatePartitionTransformEPN10impala_udf15FunctionContextERKNS1_6IntValES6_'],
  [['iceberg_truncate_transform'], 'BIGINT', ['BIGINT', 'BIGINT'],
      '_ZN6impala16IcebergFunctions26TruncatePartitionTransformEPN10impala_udf15FunctionContextERKNS1_9BigIntValES6_'],
  [['iceberg_truncate_transform'], 'DECIMAL', ['DECIMAL', 'INT'],
      '_ZN6impala16IcebergFunctions26TruncatePartitionTransformEPN10impala_udf15FunctionContextERKNS1_10DecimalValERKNS1_6IntValE'],
  [['iceberg_truncate_transform'], 'DECIMAL', ['DECIMAL', 'BIGINT'],
      '_ZN6impala16IcebergFunctions26TruncatePartitionTransformEPN10impala_udf15FunctionContextERKNS1_10DecimalValERKNS1_9BigIntValE'],
  [['iceberg_truncate_transform'], 'STRING', ['STRING', 'INT'],
      '_ZN6impala16IcebergFunctions26TruncatePartitionTransformEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_6IntValE'],
  [['iceberg_bucket_transform'], 'INT', ['INT', 'INT'],
      '_ZN6impala16IcebergFunctions24BucketPartitionTransformEPN10impala_udf15FunctionContextERKNS1_6IntValES6_'],
  [['iceberg_bucket_transform'], 'INT', ['BIGINT', 'INT'],
      '_ZN6impala16IcebergFunctions24BucketPartitionTransformEPN10impala_udf15FunctionContextERKNS1_9BigIntValERKNS1_6IntValE'],
  [['iceberg_bucket_transform'], 'INT', ['DECIMAL', 'INT'],
      '_ZN6impala16IcebergFunctions24BucketPartitionTransformEPN10impala_udf15FunctionContextERKNS1_10DecimalValERKNS1_6IntValE'],
  [['iceberg_bucket_transform'], 'INT', ['STRING', 'INT'],
      '_ZN6impala16IcebergFunctions24BucketPartitionTransformEPN10impala_udf15FunctionContextERKNS1_9StringValERKNS1_6IntValE'],
  [['iceberg_bucket_transform'], 'INT', ['DATE', 'INT'],
      '_ZN6impala16IcebergFunctions24BucketPartitionTransformEPN10impala_udf15FunctionContextERKNS1_7DateValERKNS1_6IntValE'],
  [['iceberg_bucket_transform'], 'INT', ['TIMESTAMP', 'INT'],
      '_ZN6impala16IcebergFunctions24BucketPartitionTransformEPN10impala_udf15FunctionContextERKNS1_12TimestampValERKNS1_6IntValE'],
]



### `common/function-registry/` 目录整体作用

这个目录是 Impala **内置函数（built-in functions）注册系统的核心**，负责：
- 定义所有 Impala 支持的 SQL 内置函数（数学、字符串、日期、条件、哈希、掩码等）。
- **自动生成** 前端（FE, Java）和后端（BE, C++）需要的注册代码。
- 确保新增内置函数只需修改 Python 数据文件（`impala_functions.py`），无需手动改 Java 或 C++ 代码。

这是 Impala 维护内置函数的**中心化、可扩展**设计，避免了手动同步 FE/BE 的繁琐工作。

### 文件逐个详细解释

#### 1. `.gitignore`
```ignore
*.pyc
generated_functions.py
```
- 忽略 Python 字节码文件和可能的临时生成文件。
- `generated_functions.py` 可能曾是旧版本的生成产物，现在已不再使用（当前生成的是 Java 文件）。

#### 2. `CMakeLists.txt`
```cmake
set(BE_OUTPUT_DIR ...)          # 后端生成路径（未实际使用）
set(FE_OUTPUT_DIR ...)          # 前端 Java 生成路径

set(CODE_GEN_OUTPUT ...)        # 要生成的 Java 文件：ScalarBuiltins.java

set(GEOSPATIAL_CODE_GEN_OUTPUT ...)  # 地理空间 wrapper 文件列表

set(FUNCTION_REGISTRY_INPUT ...)     # 普通内置函数生成依赖的 Python 文件
set(GEOSPATIAL_WRAPPER_INPUT ...)    # 地理空间 wrapper 生成依赖

add_custom_command(OUTPUT ${CODE_GEN_OUTPUT} COMMAND ./gen_builtins_catalog.py ...)
add_custom_command(OUTPUT ${GEOSPATIAL_CODE_GEN_OUTPUT} COMMAND ./gen_geospatial_udf_wrappers.py ...)

add_custom_target(function-registry ALL DEPENDS ${CODE_GEN_OUTPUT})

if($ENV{USE_APACHE_HIVE_3} STREQUAL "false")
  add_custom_target(geospatial-udf-wrappers ALL DEPENDS ${GEOSPATIAL_CODE_GEN_OUTPUT})
  add_dependencies(geospatial-udf-wrappers function-registry)
else()
  add_custom_target(geospatial-udf-wrappers)  # 空目标，跳过生成
endif()
```

**核心作用**：
- 在构建时自动运行两个 Python 脚本生成代码。
- 生成的文件被后续 Java 编译（前端）使用。

**关键目标**：
- `function-registry`：生成所有普通内置函数的 FE 注册代码（`ScalarBuiltins.java`），**每次构建必执行**（`ALL`）。
- `geospatial-udf-wrappers`：
  - 当使用 CDP Hive（非 Apache Hive 3）时，生成 ESRI 地理空间 UDF 的 Java wrapper（解决 varargs 兼容问题）。
  - 当使用 Apache Hive 3 时，**跳过生成**（因为 Hive 3 已自带兼容实现）。

#### 3. `gen_builtins_catalog.py`
```python
# 从 impala_functions.py 读取函数元数据
# 生成 FE 的 ScalarBuiltins.java，里面调用 db.addScalarBuiltin(...) 注册所有内置函数
```

**工作流程**：
1. 导入 `impala_functions.py`，读取 `visible_functions` 和 `invisible_functions`。
2. 为每个函数生成一行 Java 注册代码：
   ```java
   db.addScalarBuiltin("fn_name", "symbol", user_visible, ...);
   ```
3. 输出到 `fe/generated-sources/.../ScalarBuiltins.java`。

**为什么需要生成？**
- 前端（Java）需要在启动时向 Catalog 注册所有内置函数（函数名、返回类型、参数类型、后端符号、是否可见等）。
- 手动维护几百个函数的注册代码极易出错，通过脚本保证 FE/BE 完全一致。

#### 4. `gen_geospatial_udf_wrappers.py`
```python
# 为 Hive ESRI 地理空间 UDF（如 ST_ConvexHull、ST_LineString 等）生成 Java wrapper 类
# 目的：桥接 Hive 的 varargs UDF 与 Impala 不支持 varargs 的限制
```

**问题背景**：
- Hive 的 ESRI 地理函数（如 `ST_ConvexHull(Geometry, ...)`）使用 Java varargs。
- Impala 的 UDF 调用机制不支持 varargs，需要固定参数个数。
- 解决方案：生成多个固定参数个数的 wrapper 方法，内部调用原 varargs 方法。

**生成示例**（以 ST_ConvexHull 为例）：
```java
public class ST_ConvexHull_Wrapper extends ST_ConvexHull {
  public BytesWritable evaluate(BytesWritable arg0) { return super.evaluate(arg0); }
  public BytesWritable evaluate(BytesWritable arg0, BytesWritable arg1) { return super.evaluate(arg0, arg1); }
  // ... 最多到 8 个参数
}
```

**为什么条件生成？**
- CDP 版 Hive 自带这些 UDF，但 varargs 不兼容 Impala → 需要 wrapper。
- Apache Hive 3 已修复或提供兼容方式 → 不需要 wrapper（避免冲突）。

#### 5. `impala_functions.py`
```python
# 核心数据文件：定义所有内置函数的元数据
visible_functions = [ ... ]   # 用户可见的函数（如 abs、pi、substr 等）
invisible_functions = [ ... ] # 内部函数（如某些分区变换函数）
```

**每条记录格式**：
```python
[sql_names], return_type, [args], backend_symbol, [prepare_symbol], [close_symbol]
```
示例：
```python
[['abs'], 'DOUBLE', ['DOUBLE'], 'impala::UdfBuiltins::Abs'],
[['now', 'current_timestamp'], 'TIMESTAMP', [], '_ZN6impala18TimestampFunctions3NowEPN10impala_udf15FunctionContextE'],
[['least'], 'STRING', ['STRING', '...'], '...']  # 支持 varargs
```

**这是添加新内置函数的唯一入口**：
1. 在这里添加一条记录。
2. 在后端实现对应的 C++ 函数（符号匹配）。
3. 重新构建 → 自动生成 FE 注册代码。

### 总结：整个机制的优势

| 优势                  | 说明                                                                 |
|-----------------------|----------------------------------------------------------------------|
| **中心化维护**        | 所有内置函数信息集中在一份 Python 文件中，便于查看和修改。          |
| **自动同步 FE/BE**    | 生成的 Java 代码保证前端注册与后端实现一致，避免手动错误。          |
| **支持复杂特性**      | varargs、prepare/close 函数、可见性控制、地理函数 wrapper 等。      |
| **条件生成**          | 根据 Hive 版本（CDP vs Apache）智能决定是否生成地理 wrapper。       |
| **易扩展**            | 新增函数只需改 `impala_functions.py` + 后端实现，无需改 Java。      |

这个设计是 Impala 能够支持数百个内置函数且保持一致性的关键所在，也是其可维护性的重要体现。
**********************************************************************************
请介绍下gen_builtins_catalog.py的内容：

```python
#!/usr/bin/env python3
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

# This script generates the FE calls to populate the builtins.
# To add a builtin, add an entry to impala_functions.py.

from __future__ import absolute_import, division, print_function
import os
import impala_functions

java_registry_preamble = '\
// \n\
//  Licensed under the Apache License, Version 2.0 (the "License");\n\
//  you may not use this file except in compliance with the License.\n\
//  You may obtain a copy of the License at\n\
// \n\
//  http://www.apache.org/licenses/LICENSE-2.0\n\
// \n\
//  Unless required by applicable law or agreed to in writing, software\n\
//  distributed under the License is distributed on an "AS IS" BASIS,\n\
//  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.\n\
//  See the License for the specific language governing permissions and\n\
//  limitations under the License.\n\
\n\
// This is a generated file, DO NOT EDIT.\n\
// To add new functions, see the generator at\n\
// common/function-registry/gen_builtins_catalog.py or the function list at\n\
// common/function-registry/impala_functions.py.\n\
\n\
package org.apache.impala.builtins;\n\
\n\
import org.apache.impala.catalog.Type;\n\
import org.apache.impala.catalog.Db;\n\
\n\
public class ScalarBuiltins { \n\
  public static void initBuiltins(Db db) { \
\n'

java_registry_epilogue = '\
  }\n\
}\n'

FE_PATH = os.path.expandvars(
  "$IMPALA_HOME/fe/generated-sources/gen-java/org/apache/impala/builtins/")

# This contains all the metadata to describe all the builtins.
# Each meta data entry is itself a map to store all the meta data
#   - fn_name, ret_type, args, symbol, sql_names
meta_data_entries = []


# Read in the function and add it to the meta_data_entries map
def add_function(fn_meta_data, user_visible):
  assert 4 <= len(fn_meta_data) <= 6, \
         "Invalid function entry in impala_functions.py:\n\t" + repr(fn_meta_data)
  entry = {}
  entry["sql_names"] = fn_meta_data[0]
  entry["ret_type"] = fn_meta_data[1]
  entry["args"] = fn_meta_data[2]
  entry["symbol"] = fn_meta_data[3]
  if len(fn_meta_data) >= 5:
    entry["prepare"] = fn_meta_data[4]
  if len(fn_meta_data) >= 6:
    entry["close"] = fn_meta_data[5]
  entry["user_visible"] = user_visible
  meta_data_entries.append(entry)


def generate_fe_entry(entry, name):
  java_output = ""
  java_output += "\"" + name + "\""
  java_output += ", \"" + entry["symbol"] + "\""
  if entry["user_visible"]:
    java_output += ", true"
  else:
    java_output += ", false"

  if 'prepare' in entry:
    java_output += ', "%s"' % entry["prepare"]
    if 'close' in entry:
      java_output += ', "%s"' % entry["close"]
    else:
      java_output += ', null'

  # Check the last entry for varargs indicator.
  if entry["args"] and entry["args"][-1] == "...":
    entry["args"].pop()
    java_output += ", true"
  else:
    java_output += ", false"

  java_output += ", Type." + entry["ret_type"]
  for arg in entry["args"]:
    java_output += ", Type." + arg
  return java_output


# Generates the FE builtins init file that registers all the builtins.
def generate_fe_registry_init(filename):
  java_registry_file = open(filename, "w")
  java_registry_file.write(java_registry_preamble)

  for entry in meta_data_entries:
    for name in entry["sql_names"]:
      java_output = generate_fe_entry(entry, name)
      java_registry_file.write("    db.addScalarBuiltin(%s);\n" % java_output)

  java_registry_file.write("\n")
  java_registry_file.write(java_registry_epilogue)
  java_registry_file.close()


if __name__ == "__main__":
  # Read the function metadata inputs
  for function in impala_functions.visible_functions:
    add_function(function, True)
  for function in impala_functions.invisible_functions:
    add_function(function, False)

  if not os.path.exists(FE_PATH):
    os.makedirs(FE_PATH)

  generate_fe_registry_init(FE_PATH + "ScalarBuiltins.java")
```
****************************************************************************
### `gen_builtins_catalog.py` 文件整体介绍

这是一个 **Python 生成脚本**，位于 Impala 源码的 `common/function-registry/` 目录下。

**核心作用**：  
从 `impala_functions.py` 中读取所有内置函数（built-in functions）的元数据，**自动生成前端（Frontend，Java 代码）注册这些函数所需的 Java 文件** —— 具体是 `ScalarBuiltins.java`。

Impala 前端在启动时需要向 Catalog（目录服务）注册所有内置标量函数（scalar functions），包括函数名、返回类型、参数类型、后端符号、是否对用户可见、是否需要 prepare/close 函数、是否支持可变参数（varargs）等。这个注册过程如果手动维护会非常繁琐且容易出错，因此使用脚本自动生成，确保前后端完全一致。

**生成的文件路径**：  
`$IMPALA_HOME/fe/generated-sources/gen-java/org/apache/impala/builtins/ScalarBuiltins.java`

**关键特点**：
- 生成的文件明确标注 **“This is a generated file, DO NOT EDIT.”**（这是生成文件，请勿编辑）。
- 新增或修改内置函数只需修改 `impala_functions.py`，重新构建即可自动更新 Java 注册代码。

### 脚本逐段分解释

#### 1. 头部与导入
```python
#!/usr/bin/env python3
...
from __future__ import absolute_import, division, print_function
import os
import impala_functions
```
- 使用 Python 3。
- 导入 `impala_functions.py`，这是所有内置函数元数据的**唯一来源**。

#### 2. Java 文件的前置和后置代码
```python
java_registry_preamble = '...'
java_registry_epilogue = '...'
```
- `preamble`：Java 文件的开头，包括版权声明、包名、导入、类定义和 `initBuiltins` 方法的开始部分。
- `epilogue`：方法和类的结束部分。
- 生成的 Java 类结构大致为：
  ```java
  public class ScalarBuiltins {
    public static void initBuiltins(Db db) {
      // 这里插入大量 db.addScalarBuiltin(...) 调用
    }
  }
  ```

#### 3. 输出路径
```python
FE_PATH = os.path.expandvars(
  "$IMPALA_HOME/fe/generated-sources/gen-java/org/apache/impala/builtins/")
```
- 计算前端生成的 Java 文件目录。
- 如果目录不存在，主函数中会自动创建。

#### 4. 全局元数据容器
```python
meta_data_entries = []
```
- 用于存储所有解析后的函数元数据（字典形式）。

#### 5. `add_function` 函数
```python
def add_function(fn_meta_data, user_visible):
  ...
```
- 将 `impala_functions.py` 中的一条函数记录解析为结构化字典。
- 支持 4~6 个字段（返回类型、参数列表、后端符号、可选 prepare/close 函数）。
- 添加 `user_visible` 标志（区分用户可见函数和内部函数）。

#### 6. `generate_fe_entry` 函数（核心生成逻辑）
```python
def generate_fe_entry(entry, name):
  ...
```
- 为一个具体的 SQL 函数名（如 "abs" 或 "lower"）生成一行 `db.addScalarBuiltin(...)` 的参数字符串。
- 处理细节：
  - 函数名、后端符号。
  - 是否对用户可见（true/false）。
  - prepare/close 函数符号（如果有）。
  - 是否支持 varargs（参数列表末尾为 "..." 时设为 true）。
  - 返回类型和所有参数类型（转换为 `Type.XXX` 形式）。

示例输出（简化）：
```java
db.addScalarBuiltin("abs", "impala::UdfBuiltins::Abs", true, false, Type.DOUBLE, Type.DOUBLE);
```

#### 7. `generate_fe_registry_init` 函数
```python
def generate_fe_registry_init(filename):
  ...
```
- 写入完整的 Java 文件：
  1. 先写 preamble。
  2. 遍历所有函数元数据，对每个 SQL 别名（如 ['lower', 'lcase']）生成一行注册调用。
  3. 最后写 epilogue。

#### 8. 主函数 `__main__`
```python
if __name__ == "__main__":
  # 读取 visible 和 invisible 函数
  for function in impala_functions.visible_functions:
    add_function(function, True)
  for function in impala_functions.invisible_functions:
    add_function(function, False)

  # 创建输出目录（如不存在）
  if not os.path.exists(FE_PATH):
    os.makedirs(FE_PATH)

  # 生成 Java 文件
  generate_fe_registry_init(FE_PATH + "ScalarBuiltins.java")
```
- 加载所有函数（可见 + 不可见）。
- 确保输出目录存在。
- 执行生成。

### 生成的 Java 文件作用

生成的 `ScalarBuiltins.java` 中的 `initBuiltins(Db db)` 方法会在 Impala Frontend 启动时被调用，将几百个内置函数注册到 Catalog 中，从而：
- 支持 SQL 查询使用这些函数。
- 提供函数签名用于查询规划和类型检查。
- 区分用户可见函数（`SHOW FUNCTIONS` 显示）和内部函数（隐藏）。

### 总结优势

| 优势                  | 说明                                                                 |
|-----------------------|----------------------------------------------------------------------|
| **单一真相来源**      | 所有函数信息只在 `impala_functions.py` 维护，避免前后端不一致。       |
| **自动生成**          | 新增函数只需改 Python 文件，构建时自动更新 Java 注册代码。           |
| **支持复杂特性**      | varargs、prepare/close 函数、用户可见性等全部自动化处理。            |
| **易维护**            | 开发者无需懂 Java 注册细节，只关注函数元数据和后端实现。             |

这个脚本是 Impala 内置函数系统**高度自动化和可维护性**的关键体现，是大型分析引擎常见的优秀工程实践。
********************************************************************************
我没有看懂你提到的；“这是临时方案（注释中有 TODO），因为旧版 ComputeFunctions 使用模板实例化。”是不是和“

```python

# Generates the BE symbol for the Compute Function class_name::fn_name<templated_type>.
# Does not handle varargs.
# TODO: this is a stopgap. ComputeFunctions are being removed and we can use the
# symbol lookup code in the BE.”相关
```
是的，你完全看懂了！这两句话指的正是同一件事。

让我详细解释一下这个注释的背景和含义：

### 注释原文
```python
# Generates the BE symbol for the Compute Function class_name::fn_name<templated_type>.
# Does not handle varargs.
# TODO: this is a stopgap. ComputeFunctions are being removed and we can use the
# symbol lookup code in the BE.
```

### 解释

1. **“ComputeFunctions” 是什么？**
   - 在 Impala 的早期版本（大约 2015–2020 年左右），很多内置标量函数（如加减乘除、比较、cast 等）是通过一个名为 `ComputeFunctions` 的 **C++ 模板类** 实现的。
   - 例如：`ComputeFunctions::Add<int>(...)`、`ComputeFunctions::Add<double>(...)` 等。
   - 这些函数通过模板实例化生成不同类型版本，符号（mangled name）非常复杂，需要像 `templated_type_symbol_map` 这样的映射表来手动构造。

2. **为什么需要 `symbol()` 这个辅助函数？**
   - 因为前端（Java）需要知道后端精确的 mangled symbol 才能正确注册函数。
   - 所以 `impala_functions.py` 中提供了 `symbol()` 函数来**手动生成**这些模板函数的符号。
   - 这就是注释说的 “Generates the BE symbol for the Compute Function...”。

3. **为什么说这是“临时方案”（stopgap）？**
   - Impala 开发团队早已决定**逐步移除 ComputeFunctions** 这个旧机制。
   - 取而代之的是：每个函数都有自己明确的、非模板的 C++ 实现函数（例如 `impala::MathFunctions::Add`、`impala::StringFunctions::Substring` 等）。
   - 新机制下，函数符号是**普通函数名**，更容易维护，也不需要手动 mangling。
   - 后端（BE）已经实现了**符号查找机制**（symbol lookup），前端可以通过函数名直接找到对应实现，无需硬编码 mangled 符号。

4. **TODO 的含义**
   - 一旦所有旧的 `ComputeFunctions` 模板函数都被迁移到新实现，这个 `symbol()` 函数和 `templated_type_symbol_map` 就可以彻底删除了。
   - `impala_functions.py` 中的函数条目也将全部使用**人类可读的符号**（如 `'impala::MathFunctions::Add'`），而不是一长串 `_ZN...` 的 mangled 名。

### 当前状态（2026 年 1 月）

- 在你提供的这个版本（较旧的 Cloudera 分支）中，仍然保留了部分旧的 ComputeFunctions 模板函数，所以 `symbol()` 和映射表还在使用。
- 但在最新的 Apache Impala master 分支（截至 2026 年），**绝大部分甚至全部 ComputeFunctions 已被移除**，`impala_functions.py` 中几乎所有函数都使用清晰的 `impala::XXX::FunctionName` 形式符号。
- 这个 TODO 实际上已经接近完成或已完成（取决于具体分支）。

### 总结

你提到的那句话：
> “这是临时方案（注释中有 TODO），因为旧版 ComputeFunctions 使用模板实例化。”

**完全正确**，它就是对文件中这个 TODO 注释的准确翻译和理解。

这个机制是 Impala 从早期模板重度依赖的架构，向更现代、易维护的普通函数实现迁移过程中的“历史遗留”。一旦迁移完成，函数注册会变得更简单、更安全。
****************************************************************************

请问这段注释是什么含义：

```python
# The format is:
#   [sql aliases], <return_type>, [<args>], <backend symbol>,
# With an optional
#   <prepare symbol>, <close symbol>
#
# 'sql aliases' are the function names that can be used from sql. There must be at least
# one per function.
#
# The symbol can be empty for functions that are not yet implemented or are special-cased
# in Expr::CreateExpr() (i.e., functions that are implemented via a custom Expr class
# rather than a single function).
```

这段注释是 `impala_functions.py` 文件中对**每条内置函数记录格式**的详细说明，帮助开发者正确添加或修改函数条目。

下面逐行解释它的含义：

```python
# The format is:
#   [sql aliases], <return_type>, [<args>], <backend symbol>,
# With an optional
#   <prepare symbol>, <close symbol>
```

**每条函数记录是一个 Python 列表（list），最多包含 6 个元素**：

1. **[sql aliases]**（必填）  
   - 一个字符串列表，里面是这个函数在 SQL 中可以使用的名字（别名）。  
   - 必须至少有一个名字。  
   - 示例：`['now', 'current_timestamp']` → SQL 中既可以用 `now()` 也可以用 `current_timestamp()` 调用同一个函数。

2. **<return_type>**（必填）  
   - 函数返回值的类型，用字符串表示（如 `'DOUBLE'`、`'STRING'`、`'TIMESTAMP'`、`'BIGINT'` 等）。

3. **[<args>]**（必填）  
   - 参数类型列表，每个元素是一个字符串。  
   - 空列表 `[]` 表示无参数函数。  
   - 如果支持可变参数（varargs），在列表最后加一个 `'...'`，如 `['STRING', '...']` 表示 `concat(str1, str2, ...)`。

4. **<backend symbol>**（必填，但可以为空字符串）  
   - 后端（Backend，C++ 代码）实现的函数符号。  
   - 通常是 C++ 的 mangled name（名称修饰后的符号），如 `'_ZN6impala18TimestampFunctions3NowE...'`，或者人类可读的形式如 `'impala::MathFunctions::Pi'`。  
   - **可以为空字符串 `''`**，含义是：
     - 该函数尚未在后端实现（占位）。
     - 或者该函数是**特殊处理**的：Impala 在解析 SQL 时，会在 `Expr::CreateExpr()` 中用自定义的 Expr 子类（而不是单个普通函数）来实现它。  
       典型例子：`if(condition, a, b)`、`coalesce(...)`、`isnull(a, b)` 等条件函数，它们在后端不是一个简单函数调用，而是生成特定的 `CaseExpr` 或 `IsNullExpr` 节点。

5. **<prepare symbol>**（可选）  
   - 如果函数是有状态的（需要初始化资源，如正则表达式编译、随机种子等），这里填写后端的 Prepare 函数符号。  
   - 只在第 5 个位置出现时有效。

6. **<close symbol>**（可选）  
   - 与 prepare 对应，用于释放资源。  
   - 只在第 6 个位置出现时有效。

### 示例说明

```python
[['now', 'current_timestamp'], 'TIMESTAMP', [], '_ZN6impala18TimestampFunctions3NowEPN10impala_udf15FunctionContextE'],
```
- SQL 别名：now / current_timestamp  
- 返回 TIMESTAMP  
- 无参数  
- 后端普通函数实现（有 mangled symbol）

```python
[['if'], 'BOOLEAN', ['BOOLEAN', 'BOOLEAN', 'BOOLEAN'], ''],
```
- SQL 别名：if  
- 返回 BOOLEAN  
- 3 个 BOOLEAN 参数  
- 后端符号为空 → 表示由 `Expr::CreateExpr()` 特殊处理（生成 CaseExpr，而不是调用单个函数）

```python
[['rand','random'], 'DOUBLE', [], 'impala::MathFunctions::Rand',
 '_ZN6impala13MathFunctions11RandPrepare...',
 '_ZN6impala13MathFunctions9RandClose...'],
```
- 有状态函数：需要 prepare 和 close 来管理随机种子

### 总结

这条注释告诉你：**要添加一个新的内置函数，只需要按照这个固定格式在 `visible_functions` 或 `invisible_functions` 列表中加一行**。最重要的是提供正确的 SQL 名字、类型签名，以及（如果已实现）对应的后端 C++ 函数符号。如果函数比较特殊（像 `if`、`coalesce`），可以暂时留空 symbol，后端会自动用自定义逻辑处理。

这样设计的好处是：所有函数信息集中在一处，前端通过脚本自动生成注册代码，保证前后端完全一致。
**********************************************************************
请问你知道：“后端（Backend，C++ 代码）实现的函数符号。”是在apache impala的哪个文件中实现的吗？
“后端（Backend，C++ 代码）实现的函数符号”（即 `impala_functions.py` 中每个函数条目第4个字段的 `<backend symbol>`）对应的实际 C++ 实现**分散在多个文件中**，位于 Impala 源码的 `be/src/exprs/` 目录下（be 表示 backend）。

这些文件按函数类别组织，主要包括（基于 Apache Impala 官方 master 分支，截至 2026 年 1 月）：

- **math-functions.cc / math-functions-ir.cc**  
  实现数学函数（如 `pi()`、`abs()`、`sin()`、`pow()`、`rand()`、`least()`、`greatest()` 等）。  
  类名：`impala::MathFunctions`

- **string-functions.cc / string-functions-ir.cc**  
  实现字符串函数（如 `lower()`、`substr()`、`concat()`、`regexp_like()`、`mask()` 等）。  
  类名：`impala::StringFunctions`

- **timestamp-functions.cc / timestamp-functions-ir.cc**  
  实现 TIMESTAMP 类型的时间函数（如 `now()`、`year()`、`date_add()`、`from_unixtime()` 等）。

- **date-functions.cc / date-functions-ir.cc**  
  实现 DATE 类型的时间函数（如 `current_date()`、`date_trunc()`、`months_between()` 等）。  
  类名：`impala::DateFunctions`

- **utility-functions.cc**  
  实现工具函数（如 `uuid()`、`version()`、`user()`、`sleep()`、`fnv_hash()`、`md5()` 等）。  
  类名：`impala::UtilityFunctions`

- **udf-builtins.cc / udf-builtins-ir.cc** / **udf-builtins.h**  
  实现一些早期或杂项内置函数（如 `trunc()`、`extract()`、`pi()` 的旧版等）。  
  类名：`impala::UdfBuiltins`（部分旧函数仍在这里）

- **bit-byte-functions.cc / bit-byte-functions-ir.cc**  
  实现位运算函数（如 `bitand()`、`getbit()`、`shiftleft()` 等）。  
  类名：`impala::BitByteFunctions`

- **mask-functions.cc / mask-functions.h**  
  实现数据脱敏函数（如 `mask()`、`mask_first_n()`、`mask_hash()` 等）。  
  类名：`impala::MaskFunctions`

- **其他专项文件**（如 `decimal-functions.cc`、`conditional-functions.cc`、`operators.cc` 等）  
  实现 Decimal、条件函数（`if()`、`coalesce()`）、运算符重载等。

**注意**：
- 文件名以 `-ir.cc` 结尾的是**代码生成版本**（使用 LLVM IR，支持运行时优化内联），对应非 `-ir.cc` 的解释执行版本。
- 一些特殊函数（如 `if()`、`coalesce()`）symbol 为空，因为它们在后端不是单个函数调用，而是通过 `Expr::CreateExpr()` 生成自定义 Expr 节点（如 CaseExpr）。
- 你提供的旧版本中很多 symbol 是 mangled name（如 `_ZN6impala...`），对应这些文件中的函数实现。在最新 Apache Impala 中，大多已迁移为人类可读的 `impala::XXX::FunctionName` 形式。

如果你要查找某个具体函数（如 `now()`），可以搜索源码中对应的类和函数名（如 `TimestampFunctions::Now`）。官方仓库：https://github.com/apache/impala/tree/master/be/src/exprs

*******************************************************************************************
请问你知道类似'_ZN6impala18TimestampFunctions3NowE...这样的每个函数C++名字，apache impala的开发者是如何找到（生成）然后添加到impala-functions.py中的么？
是的，Apache Impala 的开发者（尤其是早期版本或 Cloudera 分支）在添加内置函数时，获取那些像 `_ZN6impala18TimestampFunctions3NowE...` 这样的 **mangled（名称修饰）符号** 的常见方式是：

### 主要方法：使用 `nm` 命令从编译后的二进制或对象文件中提取

根据官方文档《Adding a builtin function to Impala》（Impala Confluence Wiki）中的明确提示：

> Tip: You can find the function symbol name after building your UDF by running "nm -g" on the compiled impalad binary or the .cc.o file.

**具体操作步骤**：
1. 在源码中实现函数（例如在 `timestamp-functions.cc` 中添加 `TimestampFunctions::Now`）。
2. 编译 Impala（`./buildall.sh` 或 `ninja`）。
3. 在构建目录下运行：
   ```bash
   nm -g be/build/latest/service/impalad | grep Now
   # 或针对具体对象文件
   nm -g be/build/latest/exprs/timestamp-functions.o | grep Now
   ```
4. 输出类似：
   ```
   0000000000e2f0e0 T _ZN6impala18TimestampFunctions3NowEPN10impala_udf15FunctionContextE
   ```
5. 复制这个 mangled 符号，粘贴到 `impala_functions.py` 的对应条目中。

这是最可靠、最常用的方式，因为 `nm`（GNU binutils 工具）直接从 ELF 二进制中列出外部符号。

### 其他辅助方式（较少用）

- **使用 `c++filt` 反向解码**：如果知道人类可读名，可以用 `c++filt` 猜测 mangled 名，但不精确（尤其是带模板时）。
- **旧版模板函数**：早期有些函数用 `ComputeFunctions` 模板实现，开发者用文件中的 `symbol()` 函数手动生成 mangled 名（基于 Itanium ABI 规则）。
- **现代版本（推荐）**：在最新 Apache Impala master 分支，许多函数已迁移为非模板普通函数，symbol 直接用人类可读形式如 `'impala::TimestampFunctions::Now'`，无需 mangled 名（因为后端有符号查找机制）。

### 为什么旧版本多用 mangled 名？

Impala 的 builtin 函数通过 UDF 接口实现，前端（Java）需要在 Catalog 中硬编码后端符号来注册。早期没有完善的符号查找，所以必须精确匹配 mangled 名。

现在，开发者添加新函数时优先用可读符号（尤其是 -ir.cc 文件中的代码生成函数），只有遗留旧函数才保留 mangled 名。

如果你在开发新函数，参考官方 Wiki：先实现 C++ 函数 → 编译 → 用 `nm` 取符号 → 加到 `impala_functions.py` → 运行 `make function-registry` 更新 Java 代码。


**********************************************************************
你提到：“后端（BE）已经实现了符号查找机制（symbol lookup），前端可以通过函数名直接找到对应实现，无需硬编码 mangled 符号。”请问你知道apache impala是如何实现的吗？
Apache Impala 的**符号查找机制**（symbol lookup）主要在后端（Backend，C++）实现，用于**运行时动态加载内置函数**（builtin functions）的实现，尤其支持 LLVM 代码生成（codegen）路径。

### 核心实现位置
- **主要文件**：`be/src/codegen/llvm-codegen.cc` 中的 `LlvmCodeGen::LoadFunction()` 方法。
- **相关类**：`LlvmCodeGen` 类（负责管理 LLVM IR 和函数加载）。
- **调用点**：在 `be/src/exprs/scalar-fn-call.cc` 的 `ScalarFnCall::GetCodegendComputeFnImpl()` 中调用，用于标量函数调用表达式（ScalarFnCall）。

### 工作原理
Impala 的内置函数分为两种执行路径：
- **解释执行**（interpreted）：直接调用 C++ 函数指针。
- **代码生成**（codegen，使用 LLVM）：为查询生成本地机器码，更高性能。

早期版本（包括你提供的旧版本）前端生成的 `ScalarBuiltins.java` 中硬编码 mangled symbol（如 `_ZN6impala...`），后端直接用这些字符串通过 `dlsym()`（动态链接符号查找）加载函数地址。

在现代版本（Apache Impala master，约 Impala 3.x+ 后逐步完善）：
- 对于 **代码生成路径**（大多数性能敏感的内置函数，尤其是 `-ir.cc` 文件中实现的），前端注册时使用**人类可读的函数名**（如 `'impala::MathFunctions::Abs'`）。
- 后端在需要时（查询执行计划中使用 codegen）调用 `LlvmCodeGen::LoadFunction()`：
  - 传入函数名字符串（demangled 或可读形式）、返回类型、参数类型、是否 varargs 等。
  - 该方法从预加载的 **LLVM 内置函数库**（pre-parsed Impala library functions）中查找匹配的 IR 函数。
  - 如果找到，直接克隆并优化该 IR 函数到当前查询的 LLVM module 中（无需 mangled symbol）。
  - 支持缓存（cache_entry_），避免重复加载。
- 这实现了“符号查找”：后端根据函数签名动态匹配和加载实现，无需前端硬编码 mangled 名。

### 为什么这样设计？
- 避免 ABI 兼容性问题（mangled 名随编译器版本变化）。
- 支持 IR 函数内联和优化（codegen 的核心优势）。
- 简化维护：新增函数只需在后端实现，前端用可读名注册。

### 遗留与过渡
- 一些旧函数仍使用 mangled symbol（通过 `dlsym()` 加载）。
- 特殊函数（如 `if()`、`coalesce()`）symbol 为空，由 `ScalarExprEvaluator` 或自定义 Expr 类处理。
- 文档（Adding a builtin function to Impala）建议新函数使用可读名，尤其是 IR 版本。

如果你查看最新源码（https://github.com/apache/impala），搜索 `LoadFunction` 或 `addScalarBuiltin`，可以看到这个机制的完整实现。这也是为什么新版本 `impala_functions.py` 中越来越多使用 `'impala::XXX::FunctionName'` 而非 mangled 名。
********************************************************************
我补充下：

```cpp
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.


#ifndef IMPALA_CODEGEN_LLVM_CODEGEN_H
#define IMPALA_CODEGEN_LLVM_CODEGEN_H

#include "common/status.h"

#include <map>
#include <memory>
#include <string>
#include <vector>
#include <unordered_set>
#include <boost/scoped_ptr.hpp>

#include <boost/unordered_set.hpp>

#include <llvm/IR/DerivedTypes.h>
#include <llvm/IR/IRBuilder.h>
#include <llvm/IR/Intrinsics.h>
#include <llvm/IR/LLVMContext.h>
#include <llvm/IR/Module.h>
#include <llvm/Support/raw_ostream.h>

#include "codegen/llvm-codegen-object-cache.h"
#include "exprs/scalar-expr.h"
#include "impala-ir/impala-ir-functions.h"
#include "runtime/types.h"
#include "util/runtime-profile.h"

/// Forward declare all llvm classes to avoid namespace pollution.
namespace llvm {
  class AllocaInst;
  class BasicBlock;
  class ConstantFolder;
  class DiagnosticInfo;
  class ExecutionEngine;
  class Function;
  class LLVMContext;
  class Module;
  class NoFolder;
  class PointerType;
  class StructType;
  class TargetData;
  class Type;
  class Value;
  namespace legacy {
    class FunctionPassManager;
    class PassManager;
  }

  template<typename T, typename I>
  class IRBuilder;

  class IRBuilderDefaultInserter;
}

// The number of function calls replaced is not knowable when UBSAN is enabled, since it
// can double the number of references to a function. To fix, we replaced
// "DCHECK_EQ(replaced" with "DCHECK_REPLACE_COUNT(replaced":
//
// find be/src -type f -execdir sed -i s/DCHECK_EQ\(replaced,\ /DCHECK_REPLACE_COUNT\(replaced,\ /g {} \;
#if defined(UNDEFINED_SANITIZER)
#define DCHECK_REPLACE_COUNT(p, q) DCHECK_GE(p, q); DCHECK_LE(p, 2*(q))
#else
#define DCHECK_REPLACE_COUNT(p, q) DCHECK_EQ(p, q)
#endif

namespace impala {

class CodegenCallGraph;
class CodegenFnPtrBase;
class CodegenSymbolEmitter;
class FragmentState;
class ImpalaMCJITMemoryManager;
class SubExprElimination;
class Thread;
class TupleDescriptor;
class CodeGenCache;
class CodeGenCacheKey;

/// Define builder subclass in case we want to change the template arguments later
class LlvmBuilder : public llvm::IRBuilder<> {
  using llvm::IRBuilder<>::IRBuilder;
};

/// LLVM code generator.  This is the top level object to generate jitted code.
//
/// LLVM provides a c++ IR builder interface so IR does not need to be written
/// manually.  The interface is very low level so each line of IR that needs to
/// be output maps 1:1 with calls to the interface.
/// The llvm documentation is not fantastic and a lot of this was figured out
/// by experimenting.  Thankfully, their API is pretty well designed so it's
/// possible to get by without great documentation.  The llvm tutorial is very
/// helpful, https://llvm.org/docs/tutorial/LangImpl01.html.  In this tutorial, they
/// go over how to JIT an AST for a toy language they create.
/// It is also helpful to use their online app that lets you compile c/c++ to IR.
/// http://llvm.org/demo/index.cgi.
//
/// This class provides two interfaces, one for testing and one for the query
/// engine.  The interface for the query engine will load the cross-compiled
/// IR module (output during the build) and extract all of functions that will
/// be called directly.  The test interface can be used to load any precompiled
/// module or none at all (but this class will not validate the module).
//
/// There are two classes of functions defined based on how they are generated:
/// 1. Handcrafted functions - These functions are built from scratch using the IRbuilder
/// interface.
/// 2. Cross-compiled functions - These functions are loaded directly from a
/// cross-compiled IR module and are either directly used or are cloned and modified
/// before use.
//
/// This class is not threadsafe.  During the Prepare() phase of the fragment execution,
/// nodes should codegen functions, and register those functions with AddFunctionToJit().
/// Afterward, FinalizeModule() should be called at which point all codegened functions
/// are optimized and compiled. After FinalizeModule() returns, all function pointers
/// registered with AddFunctionToJit() will be pointing to the appropriate JIT'd function.
//
/// Currently, each fragment instance  will create and initialize one of these
/// objects.  This requires loading and parsing the cross compiled modules.
/// TODO: we should be able to do this once per process and let llvm compile
/// functions from across modules.
//
/// LLVM has a nontrivial memory management scheme and objects will take
/// ownership of others. The document is pretty good about being explicit with this
/// but it is not very intuitive.
/// TODO: look into diagnostic output and debuggability
/// TODO: confirm that the multi-threaded usage is correct
//
/// Function objects in the module are materialized lazily to save the cost of
/// parsing IR of functions which are dead code. An unmaterialized function is similar
/// to a function declaration which only contains the function signature and needs to
/// be materialized before optimization and compilation happen if it's not dead code.
/// Materializing a function means parsing the bitcode to populate the basic blocks and
/// instructions attached to the function object. Functions reachable by the function
/// are also materialized recursively.
//
/// Memory used for codegen is tracked via the MemTracker hierarchy. Codegen can use
/// significant memory for the IR module and for the optimization and compilation
/// algorithms. LLVM provides no way to directly track this transient memory - instead
/// the memory consumption is estimated based on the size of the IR module and released
/// once compilation finishes. Once compilation finishes, the size of the compiled
/// machine code is obtained from LLVM and and is tracked until the LlvmCodeGen object
/// is torn down and the compiled code is freed.
//
class LlvmCodeGen {
 public:
  /// This function must be called once per process before any llvm API calls are
  /// made.  It is not valid to call it multiple times. LLVM needs to allocate data
  /// structures for multi-threading support and to enable dynamic linking of jitted code.
  /// if 'load_backend', load the backend static object for llvm. This is needed
  /// when libfesupport.so is loaded from java. llvm will by default only look in
  /// the current object and not be able to find the backend symbols
  /// TODO: this can probably be removed after impalad refactor where the java
  /// side is not loading the be explicitly anymore.
  static Status InitializeLlvm(const char* procname = "main", bool load_backend = false);

  /// Creates a codegen instance for Impala initialized with the cross-compiled Impala IR.
  /// 'codegen' will contain the created object on success.
  /// 'parent_mem_tracker' - if non-NULL, the CodeGen MemTracker is created under this.
  /// 'id' is used for outputting the IR module for debugging.
  static Status CreateImpalaCodegen(FragmentState* state, MemTracker* parent_mem_tracker,
      const std::string& id, boost::scoped_ptr<LlvmCodeGen>* codegen);

  ~LlvmCodeGen();

  /// Releases all resources associated with the codegen object. It is invalid to call
  /// any other API methods after calling close.
  void Close();

  RuntimeProfile* runtime_profile() { return profile_; }
  RuntimeProfile::Counter* ir_generation_timer() { return ir_generation_timer_; }
  RuntimeProfile::Counter* main_thread_timer() { return main_thread_timer_; }
  RuntimeProfile::ThreadCounters* llvm_thread_counters() { return llvm_thread_counters_; }

  /// Turns on/off optimization passes
  void EnableOptimizations(bool enable);

  std::string DebugCacheEntryString(CodeGenCacheKey& key, bool is_lookup, bool debug_mode,
      bool success) const;

  /// For debugging. Returns the IR that was generated.  If full_module, the
  /// entire module is dumped, including what was loaded from precompiled IR.
  /// If false, only output IR for functions which were handcrafted.
  std::string GetIR(bool full_module) const;

  /// Utility struct that wraps a variable name and llvm type.
  struct NamedVariable {
    std::string name;
    llvm::Type* type;

    NamedVariable(const std::string& name="", llvm::Type* type = NULL) {
      this->name = name;
      this->type = type;
    }
  };

  /// Abstraction over function prototypes.  Contains helpers to build prototypes and
  /// generate IR for the types.
  class FnPrototype {
   public:
    /// Create a function prototype object, specifying the name of the function and
    /// the return type.
    FnPrototype(LlvmCodeGen* codegen, const std::string& name, llvm::Type* ret_type);

    /// Returns name of function
    const std::string& name() const { return name_; }

    /// (Re-)sets name of function
    void SetName(const std::string& name) { name_ = name; }

    /// Add argument
    void AddArgument(const NamedVariable& var) {
      args_.push_back(var);
    }

    void AddArgument(const std::string& name, llvm::Type* type) {
      args_.push_back(NamedVariable(name, type));
    }

    /// Generate LLVM function prototype.
    /// This is the canonical way to start generating a handcrafted codegen'd function.
    /// If a non-null 'builder' is passed, this function will also create the entry
    /// block, add it to the llvm module via the builder by setting the builder's insert
    /// point to the entry block, and add it to the list of functions handcrafted by
    /// impala. FinalizeFunction() must be called for any function generated this way
    /// otherwise it will be deleted during FinalizeModule().
    ///
    /// If 'params' is non-null, this function will also return the arguments values
    /// (params[0] is the first arg, etc). In that case, 'params' should be preallocated
    /// to be number of arguments
    llvm::Function* GeneratePrototype(
        LlvmBuilder* builder = nullptr, llvm::Value** params = nullptr);

   private:
    friend class LlvmCodeGen;

    LlvmCodeGen* codegen_;
    std::string name_;
    llvm::Type* ret_type_;
    std::vector<NamedVariable> args_;
  };

  /// Get host cpu attributes in format expected by EngineBuilder.
  static void GetHostCPUAttrs(std::unordered_set<std::string>* attrs);

  /// Returns whether or not this cpu feature is supported.
  static bool IsCPUFeatureEnabled(int64_t flag);

  /// Return a pointer type to 'type'
  llvm::PointerType* GetPtrType(llvm::Type* type);

  /// Return a pointer to pointer type to 'type'.
  llvm::PointerType* GetPtrPtrType(llvm::Type* type);

  /// Return a pointer to pointer type for 'name' type.
  llvm::PointerType* GetNamedPtrPtrType(const std::string& name);

  /// Returns llvm type for Impala's internal representation of this column type,
  /// i.e. the way Impala represents this type in a Tuple.
  llvm::Type* GetSlotType(const ColumnType& type);

  /// Return a pointer type to 'type' (e.g. int16_t*)
  llvm::PointerType* GetSlotPtrType(const ColumnType& type);

  /// Returns the type with 'name'.  This is used to pull types from clang
  /// compiled IR.  The types we generate at runtime are unnamed.
  /// The name is generated by the clang compiler in this form:
  /// <class/struct>.<namespace>::<class name>.  For example:
  /// "class.impala::AggregationNode"
  llvm::Type* GetNamedType(const std::string& name);

  /// Returns the pointer type of the type returned by GetNamedType(name)
  llvm::PointerType* GetNamedPtrType(const std::string& name);

  /// Template versions of GetNamed*Type functions that expect the llvm name of
  /// type T to be T::LLVM_CLASS_NAME. T must be a struct/class, so GetStructType
  /// can return llvm::StructType* to avoid casting on the caller side.
  template<class T>
  llvm::StructType* GetStructType() {
    return llvm::cast<llvm::StructType>(GetNamedType(T::LLVM_CLASS_NAME));
  }

  template<class T>
  llvm::PointerType* GetStructPtrType() { return GetNamedPtrType(T::LLVM_CLASS_NAME); }

  template<class T>
  llvm::PointerType* GetStructPtrPtrType() {
    return GetNamedPtrPtrType(T::LLVM_CLASS_NAME);
  }

  /// Alloca's an instance of the appropriate pointer type and sets it to point at 'v'
  llvm::Value* GetPtrTo(LlvmBuilder* builder, llvm::Value* v, const char* name = "");

  /// Creates a global value 'name' using constant 'ir_constant' and returns
  /// a pointer to the global value. Useful for creating constant function arguments
  /// which cannot be represented with primitive types (e.g. struct).
  llvm::Constant* ConstantToGVPtr(llvm::Type* type, llvm::Constant* ir_constant,
      const std::string& name);

  /// Creates a global value 'name' that is an array with element type 'element_type'
  /// containing 'ir_constants'. Returns a pointer to the global value, i.e. a pointer
  /// to a constant array of 'element_type'.
  llvm::Constant* ConstantsToGVArrayPtr(llvm::Type* element_type,
      llvm::ArrayRef<llvm::Constant*> ir_constants, const std::string& name);

  /// Returns reference to llvm context object.  Each LlvmCodeGen has its own
  /// context to allow multiple threads to be calling into llvm at the same time.
  llvm::LLVMContext& context() { return *context_.get(); }

  /// Returns execution engine interface
  llvm::ExecutionEngine* execution_engine() { return execution_engine_.get(); }

  /// Returns the cache which is for the execution engine to write the compiled functions
  /// to.
  CodeGenObjectCache* engine_cache() { return engine_cache_.get(); }

  /// Register a expr function with unique id.  It can be subsequently retrieved via
  /// GetRegisteredExprFn with that id.
  void RegisterExprFn(int64_t id, llvm::Function* function) {
    DCHECK(registered_exprs_map_.find(id) == registered_exprs_map_.end());
    registered_exprs_map_[id] = function;
    registered_exprs_.insert(function);
  }

  /// Returns a registered expr function for id or NULL if it does not exist.
  llvm::Function* GetRegisteredExprFn(int64_t id) {
    std::map<int64_t, llvm::Function*>::iterator it = registered_exprs_map_.find(id);
    if (it == registered_exprs_map_.end()) return NULL;
    return it->second;
  }

  /// Optimize and compile the module. This should be called after all functions to JIT
  /// have been added to the module via AddFunctionToJit(). If optimizations_enabled_ is
  /// false, the module will not be optimized before compilation. After FinalizeModule()
  /// is called, the LLVM module is destroyed and it is invalid to call any LlvmCodegen
  /// functions.
  /// During FinalizeModule(), a new module id might be assigned for caching storage and
  /// retrieval. If module_id is not nullptr, the final module id is returned.
  Status FinalizeModule(string* module_id = nullptr);

  /// Start executing 'FinalizeModule' in a separate thread and return.
  /// 'async_compile_thread_' is set to point to the new 'Thread' object.
  ///
  /// Execution of the query starts in interpreted mode in the calling thread while
  /// compilation is done in 'async_compile_thread_'. When compilation has finished
  /// function pointers that have been added via 'AddFunctionToJit' will be set to the
  /// compiled functions and the query will automatically use the codegen'd version the
  /// next time the corresponding function is called (we always check the codegen'd
  /// function pointer and fall back to interpreted mode if it is nullptr).
  ///
  /// The function pointers are atomic so no locking is needed.
  ///
  /// 'Close' calls 'Join' on '*async_compile_thread_' if it is not a nullptr.
  Status FinalizeModuleAsync(RuntimeProfile::EventSequence* event_sequence);

  /// Loads a native or IR function 'fn' with symbol 'symbol' from the builtins or
  /// an external library and puts the result in *llvm_fn. *llvm_fn can be safely
  /// modified in place, because it is either newly generated or cloned. The caller must
  /// call FinalizeFunction() on 'llvm_fn' once it is done modifying it. The function has
  /// return type 'return_type' (void if 'return_type' is NULL) and input argument types
  /// 'arg_types'. The first 'num_fixed_args' arguments are fixed arguments, and the
  /// remaining arguments are varargs. 'has_varargs' indicates whether the function
  /// accepts varargs. If 'has_varargs' is true, there must be at least one vararg. If
  /// the function is loaded from a library, 'cache_entry' is updated to point to the
  /// library containing the function. If 'cache_entry' is set to a non-NULL value by
  /// this function, the caller must call LibCache::DecrementUseCount() on it when done
  /// using the function.
  Status LoadFunction(const TFunction& fn, const std::string& symbol,
      const ColumnType* return_type, const std::vector<ColumnType>& arg_types,
      int num_fixed_args, bool has_varargs, llvm::Function** llvm_fn,
      LibCacheEntry** cache_entry);

  /// Replaces all instructions in 'caller' that call 'target_name' with a call
  /// instruction to 'new_fn'. The argument types of 'new_fn' must exactly match
  /// the argument types of the function to be replaced. Returns the number of
  /// call sites updated.
  ///
  /// 'target_name' must be a substring of the mangled symbol of the function to be
  /// replaced. This usually means that the unmangled function name is sufficient.
  ///
  /// Note that this modifies 'caller' in-place, so this should only be called on cloned
  /// functions.
  int ReplaceCallSites(llvm::Function* caller, llvm::Function* new_fn,
      const std::string& target_name);

  /// Same as ReplaceCallSites(), except replaces the function call instructions with the
  /// boolean value 'constant'.
  int ReplaceCallSitesWithBoolConst(llvm::Function* caller, bool constant,
      const std::string& target_name);

  /// Replace calls to functions in 'caller' where the callee's name has 'target_name'
  /// as a substring. Calls to functions are replaced with the value 'replacement'. The
  /// return value is the number of calls replaced.
  int ReplaceCallSitesWithValue(llvm::Function* caller, llvm::Value* replacement,
      const std::string& target_name);

  /// This function replaces calls to FunctionContextImpl::GetConstFnAttr() with constants
  /// derived from 'return_type', 'arg_types' and the runtime state 'state_'. Please note
  /// that this function only replaces call instructions inside 'fn' so to replace the
  /// call to FunctionContextImpl::GetConstFnAttr() inside the callee functions, please
  /// inline the callee functions (by annotating them with IR_ALWAYS_INLINE).
  ///
  /// TODO: implement a loop unroller (or use LLVM's) so we can use
  /// FunctionContextImpl::GetConstFnAttr() in loops
  int InlineConstFnAttrs(const FunctionContext::TypeDesc& return_type,
      const std::vector<FunctionContext::TypeDesc>& arg_types, llvm::Function* fn);

  /// Returns a copy of fn. The copy is added to the module.
  llvm::Function* CloneFunction(llvm::Function* fn);

  /// Replace all uses of the instruction 'from' with the value 'to', and delete
  /// 'from'. This is a wrapper around llvm::ReplaceInstWithValue().
  void ReplaceInstWithValue(llvm::Instruction* from, llvm::Value* to);

  /// Returns the i-th argument of fn.
  llvm::Argument* GetArgument(llvm::Function* fn, int i);

  /// Verify function. All handcrafted functions need to be finalized before being
  /// passed to AddFunctionToJit() otherwise the functions will be deleted from the
  /// module when the module is finalized. Also, all loaded functions that need to be JIT
  /// compiled after modification also need to be finalized.
  /// If the function does not verify, it returns NULL and the function will eventually
  /// be deleted in FinalizeModule(), otherwise, it returns the function object.
  llvm::Function* FinalizeFunction(llvm::Function* function);

  /// Prunes any unused functions from the module.
  void PruneModule();

  /// Adds the function to be automatically jit compiled when the codegen object is
  /// finalized. FinalizeModule() will set *fn_ptr to point to the jitted function.
  ///
  /// Pre-condition: FinalizeFunction() must have been called on the function passed to
  /// this method.
  ///
  /// Only functions registered with AddFunctionToJit() and their dependencies are
  /// compiled by FinalizeModule(): other functions are considered dead code and will
  /// be removed during optimization.
  ///
  /// This will also wrap functions returning DecimalVals in an ABI-compliant wrapper (see
  /// the comment in the .cc file for details). This is so we don't accidentally try to
  /// call non-compliant code from native code.
  void AddFunctionToJit(llvm::Function* fn, CodegenFnPtrBase* fn_ptr);

  /// This will generate a printf call instruction to output 'message' at the builder's
  /// insert point. If 'v1' is non-NULL, it will also be passed to the printf call. Only
  /// for debugging.
  void CodegenDebugTrace(LlvmBuilder* builder, const char* message,
      llvm::Value* v1 = NULL);

  /// Returns the string representation of a llvm::Value* or llvm::Type*
  template <typename T> static std::string Print(T* value_or_type) {
    std::string str;
    llvm::raw_string_ostream stream(str);
    value_or_type->print(stream);
    return str;
  }

  /// Returns the cross compiled function. 'ir_type' is an enum which is generated
  /// by gen_ir_descriptions.py. The returned function and its callee will be materialized
  /// recursively. Returns NULL if there is any error.
  ///
  /// If 'clone' is true, a clone of the function will be returned. Clones should be used
  /// iff the caller will modify the returned function so that the original unmodified
  /// function remains available. Avoid cloning if possible to reduce compilation time.
  ///
  /// TODO: Return Status instead.
  llvm::Function* GetFunction(IRFunction::Type ir_type, bool clone);

  /// Return the function with the symbol name 'symbol' from the module. The returned
  /// function and its callee will be recursively materialized. Returns NULL if there is
  /// any error.
  ///
  /// If 'clone' is true, a clone of the function will be returned. Clones should be used
  /// iff the caller will modify the returned function so that the original unmodified
  /// function remains available. Avoid cloning if possible to reduce compilation time.
  ///
  /// TODO: Return Status instead.
  llvm::Function* GetFunction(const string& symbol, bool clone);

  /// Returns the hash function with signature:
  ///   int32_t Hash(int8_t* data, int len, int32_t seed);
  /// If num_bytes is non-zero, the returned function will be codegen'd to only
  /// work for that number of bytes.  It is invalid to call that function with a
  /// different 'len'. Functions returned by these methods have already been finalized.
  llvm::Function* GetHashFunction(int num_bytes = -1);
  llvm::Function* GetFnvHashFunction(int num_bytes = -1);
  llvm::Function* GetMurmurHashFunction(int num_bytes = -1);

  /// Set the NoInline attribute on 'function' and remove the AlwaysInline and InlineHint
  /// attributes if present.
  void SetNoInline(llvm::Function* function) const;

  /// Allocate stack storage for local variables.  This is similar to traditional c, where
  /// all the variables must be declared at the top of the function.  This helper can be
  /// called from anywhere and will add a stack allocation for 'var' at the beginning of
  /// the function.  This would be used, for example, if a function needed a temporary
  /// struct allocated.  The allocated variable is scoped to the function.
  //
  /// This should always be used instead of calling LlvmBuilder::CreateAlloca directly.
  /// LLVM doesn't optimize alloca's occurring in the middle of functions very well (e.g,
  /// an alloca may end up in a loop, potentially blowing the stack).
  llvm::AllocaInst* CreateEntryBlockAlloca(llvm::Function* f, const NamedVariable& var);
  llvm::AllocaInst* CreateEntryBlockAlloca(
      const LlvmBuilder& builder, llvm::Type* type, const char* name = "");

  /// Same as above, except allocates an array of 'type' with 'num_entries' entries
  /// and alignment 'alignment'.
  llvm::AllocaInst* CreateEntryBlockAlloca(const LlvmBuilder& builder, llvm::Type* type,
      int num_entries, int alignment, const char* name = "");

  /// Utility to create two blocks in 'fn' for if/else codegen.  if_block and else_block
  /// are return parameters.  insert_before is optional and if set, the two blocks
  /// will be inserted before that block otherwise, it will be inserted at the end
  /// of 'fn'.  Being able to place blocks is useful for debugging so the IR has a
  /// better looking control flow.
  void CreateIfElseBlocks(llvm::Function* fn, const std::string& if_name,
      const std::string& else_name,
      llvm::BasicBlock** if_block, llvm::BasicBlock** else_block,
      llvm::BasicBlock* insert_before = NULL);

  // Creates a PHI node with two incoming blocks that will have the value 'value1' for the
  // incoming block 'incoming_block1' and the value 'value2' for incoming block
  // 'incoming_block2'.
  static llvm::PHINode* CreateBinaryPhiNode(LlvmBuilder* builder, llvm::Value* value1,
      llvm::Value* value2, llvm::BasicBlock* incoming_block1,
      llvm::BasicBlock* incoming_block2, const std::string& name);

  /// Returns a constant int of 'byte_size' bytes based on 'low_bits' and 'high_bits'
  /// which stand for the lower and upper 64-bits of the constant respectively. For
  /// values less than or equal to 64-bits, 'high_bits' is not used. This function
  /// can generate constant up to 128-bit wide. 'byte_size' must be power of 2.
  llvm::Constant* GetIntConstant(int byte_size, uint64_t low_bits, uint64_t high_bits);

  /// Initialise a constant global string and returns an i8* pointer to it.
  llvm::Value* GetStringConstant(LlvmBuilder* builder, const char* data, int len);
  llvm::Value* GetStringConstant(LlvmBuilder* builder, const std::string& str) {
    return GetStringConstant(builder, str.c_str(), str.size());
  }

  /// Returns true/false constants (bool type)
  llvm::Constant* true_value() { return true_value_; }
  llvm::Constant* false_value() { return false_value_; }
  llvm::Constant* null_ptr_value() { return llvm::ConstantPointerNull::get(ptr_type()); }

  /// Simple wrappers to reduce code verbosity
  llvm::Type* bool_type() { return llvm::Type::getInt1Ty(context()); }
  llvm::Type* i8_type() { return llvm::Type::getInt8Ty(context()); }
  llvm::Type* i16_type() { return llvm::Type::getInt16Ty(context()); }
  llvm::Type* i32_type() { return llvm::Type::getInt32Ty(context()); }
  llvm::Type* i64_type() { return llvm::Type::getInt64Ty(context()); }
  llvm::Type* i128_type() { return llvm::Type::getIntNTy(context(), 128); }
  llvm::Type* float_type() { return llvm::Type::getFloatTy(context()); }
  llvm::Type* double_type() { return llvm::Type::getDoubleTy(context()); }
  llvm::PointerType* ptr_type() { return ptr_type_; }
  llvm::Type* void_type() { return void_type_; }

  llvm::PointerType* i8_ptr_type() { return GetPtrType(i8_type()); }
  llvm::PointerType* i16_ptr_type() { return GetPtrType(i16_type()); }
  llvm::PointerType* i32_ptr_type() { return GetPtrType(i32_type()); }
  llvm::PointerType* i64_ptr_type() { return GetPtrType(i64_type()); }
  llvm::PointerType* float_ptr_type() { return GetPtrType(float_type()); }
  llvm::PointerType* double_ptr_type() { return GetPtrType(double_type()); }
  llvm::PointerType* ptr_ptr_type() { return GetPtrType(ptr_type_); }

  llvm::Constant* GetBoolConstant(bool val) { return val ? true_value_ : false_value_; }
  llvm::Constant* GetI8Constant(uint64_t val) {
    return llvm::ConstantInt::get(context(), llvm::APInt(8, val));
  }
  llvm::Constant* GetI16Constant(uint64_t val) {
    return llvm::ConstantInt::get(context(), llvm::APInt(16, val));
  }
  llvm::Constant* GetI32Constant(uint64_t val) {
    return llvm::ConstantInt::get(context(), llvm::APInt(32, val));
  }
  llvm::Constant* GetI64Constant(uint64_t val) {
    return llvm::ConstantInt::get(context(), llvm::APInt(64, val));
  }

  /// Load the module temporarily and populate 'symbols' with the symbols in the module.
  static Status GetSymbols(const string& file, const string& module_id,
      boost::unordered_set<std::string>* symbols);

  /// Codegen at the current builder location in function 'fn' to store the
  /// max/min('src', value in 'dst_slot_ptr') in 'dst_slot_ptr'
  void CodegenMinMax(LlvmBuilder* builder, const ColumnType& type,
      llvm::Value* dst_slot_ptr, llvm::Value* src, bool min, llvm::Function* fn);

  /// Codegen to call llvm memcpy intrinsic at the current builder location
  /// dst & src must be pointer types. size is the number of bytes to copy.
  /// No-op if size is zero.
  void CodegenMemcpy(LlvmBuilder* builder, llvm::Value* dst, llvm::Value* src, int size);
  void CodegenMemcpy(LlvmBuilder* builder, llvm::Value* dst, llvm::Value* src,
      llvm::Value* size);

  /// Codegen to call llvm memset intrinsic at the current builder location. 'dst' should
  /// be a pointer. No-op if size is zero.
  void CodegenMemset(LlvmBuilder* builder, llvm::Value* dst, int value, int size);

  /// Codegen to set all null bytes of the given tuple to 0.
  void CodegenClearNullBits(LlvmBuilder* builder, llvm::Value* tuple_ptr,
      const TupleDescriptor& tuple_desc);

  /// Codegen to call pool_val->Allocate(size_val).
  /// 'pool_val' has to be of type MemPool*.
  llvm::Value* CodegenMemPoolAllocate(LlvmBuilder* builder, llvm::Value* pool_val,
      llvm::Value* size_val, const char* name = "");

  /// Codegens IR to load array[idx] and returns the loaded value. 'array' should be a
  /// C-style array (e.g. i32*) or an IR array (e.g. [10 x i32]). This function does not
  /// do bounds checking.
  llvm::Value* CodegenArrayAt(
      LlvmBuilder*, llvm::Value* array, int idx, const char* name = "");

  /// Codegens IR to call the function corresponding to 'ir_type' with argument 'args'
  /// and returns the value.
  llvm::Value* CodegenCallFunction(LlvmBuilder* builder, IRFunction::Type ir_type,
      llvm::ArrayRef<llvm::Value*> args, const char* name);

  /// If there are more than this number of expr trees (or functions that evaluate
  /// expressions), avoid inlining avoid inlining for the exprs exceeding this threshold.
  static const int CODEGEN_INLINE_EXPRS_THRESHOLD = 100;

  /// If there are more than this number of expr trees (or functions that evaluate
  /// expressions), avoid inlining the function that evaluates the expression batch
  /// into the calling function.
  static const int CODEGEN_INLINE_EXPR_BATCH_THRESHOLD = 25;

  /// Name prefix of the thread counters that track async codegen time.
  static const std::string ASYNC_CODEGEN_THREAD_COUNTERS_PREFIX;

 private:
  friend class ExprCodegenTest;
  friend class LlvmCodeGenTest;
  friend class LlvmCodeGenTest_CpuAttrWhitelist_Test;
  friend class LlvmCodeGenTest_HashTest_Test;
  friend class LlvmOptTest;
  friend class SubExprElimination;
  friend class CodeGenCache;
  friend class LlvmCodeGenCacheTest;

  /// Top level codegen object. 'module_id' is used for debugging when outputting the IR.
  LlvmCodeGen(FragmentState* state, ObjectPool* pool, MemTracker* parent_mem_tracker,
      const std::string& module_id);

  /// Initializes the jitter and execution engine with the given module.
  Status Init(std::unique_ptr<llvm::Module> module);

  /// Creates a LlvmCodeGen instance initialized with the module bitcode from 'file'.
  /// 'codegen' will contain the created object on success. The functions in the module
  /// are materialized lazily. Getting a reference to a function via GetFunction() will
  /// materialize the function and its callees recursively.
  static Status CreateFromFile(FragmentState* state, ObjectPool* pool,
      MemTracker* parent_mem_tracker, const std::string& file,
      const std::string& id, boost::scoped_ptr<LlvmCodeGen>* codegen);

  /// Creates a LlvmCodeGen instance initialized with the module bitcode in memory.
  /// 'codegen' will contain the created object on success. The functions in the module
  /// are materialized lazily. Getting a reference to a function via GetFunction() will
  /// materialize the function and its callees recursively.
  static Status CreateFromMemory(FragmentState* state, ObjectPool* pool,
      MemTracker* parent_mem_tracker, const std::string& id,
      boost::scoped_ptr<LlvmCodeGen>* codegen);

  /// Loads an LLVM module from 'file' which is the local path to the LLVM bitcode file.
  /// The functions in the module are materialized lazily. Getting a reference to the
  /// function via GetFunction() will materialize the function and its callees
  /// recursively. The caller is responsible for cleaning up the module.
  Status LoadModuleFromFile(const string& file, std::unique_ptr<llvm::Module>* module);

  /// Loads an LLVM module. 'module_ir_buf' is the memory buffer containing LLVM bitcode.
  /// 'module_name' is the name of the module to use when reporting errors. The caller is
  /// responsible for cleaning up 'module'. The functions in the module aren't
  /// materialized. Getting a reference to the functiom via GetFunction() will materialize
  /// the function and its callees recursively.
  Status LoadModuleFromMemory(std::unique_ptr<llvm::MemoryBuffer> module_ir_buf,
      const std::string& module_name, std::unique_ptr<llvm::Module>* module);

  /// Loads a module at 'file' and links it to the module associated with this
  /// LlvmCodeGen object. The 'file' must be on the local filesystem.
  Status LinkModuleFromLocalFs(const std::string& file);

  /// Same as 'LinkModuleFromLocalFs', but takes an hdfs file location instead and makes
  /// sure that the same hdfs file is not linked twice. The mtime is used ensure that the
  /// cached hdfs_file that's used is the most recent.
  Status LinkModuleFromHdfs(const std::string& hdfs_file, const time_t mtime);

  /// Strip global constructors and destructors from an LLVM module. We never run them
  /// anyway (they must be explicitly invoked) so it is dead code.
  static void StripGlobalCtorsDtors(llvm::Module* module);

  /// Set the "target-cpu" and "target-features" of 'function' to match the host's CPU's
  /// features. Having consistent attributes for all materialized functions allows
  /// generated IR to be inlined into cross-compiled functions' IR and vice versa.
  static void SetCPUAttrs(llvm::Function* function);

  /// If a symbol emitter is needed, creates one and registers it as a listener of
  /// 'execution_engine'. It is used to generate perf symbol map or disassembly.
  /// If no symbol emitter is needed, returns NULL.
  std::unique_ptr<CodegenSymbolEmitter> SetupSymbolEmitter(
      llvm::ExecutionEngine* execution_engine);

  /// Load the intrinsics impala needs.  This is a one time initialization.
  /// Values are stored in 'llvm_intrinsics_'
  Status LoadIntrinsics();

  /// Internal function for unit tests: skips Impala-specific wrapper generation logic.
  void AddFunctionToJitInternal(llvm::Function* fn, CodegenFnPtrBase* fn_ptr);

  /// Verifies the function, e.g., checks that the IR is well-formed.  Returns false if
  /// function is invalid.
  bool VerifyFunction(llvm::Function* function);

  // Used for testing.
  void ResetVerification() { is_corrupt_ = false; }

  // Lookup the codegen functions from the cache to reduce optimization time.
  bool LookupCache(CodeGenCacheKey& key);

  // Store the codegen functions to the cache if codegen cache is enabled.
  Status StoreCache(CodeGenCacheKey& key);

  /// Optimizes the module. This includes pruning the module of any unused functions.
  Status OptimizeModule();

  /// Points the function pointers in 'fns_to_jit_compile_' to the compiled functions. If
  /// 'cache' and 'cache_key' are non-NULL, retrieves the functions from the cached
  /// execution engine, otherwise from the current execution engine.
  /// Note: either both or none of 'cache' and 'cache_key' should be NULL.
  bool SetFunctionPointers(CodeGenCache* cache = nullptr,
      const CodeGenCacheKey* cache_key = nullptr);

  /// Clears generated hash fns.  This is only used for testing.
  void ClearHashFns();

  /// Finds call instructions in 'caller' where 'target_name' is a substring of the
  /// callee's name. Found instructions are appended to the 'results' vector.
  static void FindCallSites(llvm::Function* caller, const std::string& target_name,
      std::vector<llvm::CallInst*>* results);

  /// This function parses the function body of the given function 'fn' and materializes
  /// any functions called by it.
  Status MaterializeCallees(llvm::Function* fn);

  /// This is the workhorse for materializing function 'fn'. It's invoked by
  /// MaterializeFunction(). Calls LLVM to materialize 'fn' if it's materializable
  /// (i.e. the function has a definition in the module and it's not materialized yet).
  /// This function parses the bitcode of 'fn' to populate basic blocks, instructions
  /// and other data structures attached to the function object. Return error status
  /// for any error.
  Status MaterializeFunction(llvm::Function* fn);

  /// Materialize the module owned by this codegen object. This will materialize all
  /// functions and delete the module's materializer. Returns error status for any error.
  Status MaterializeModule();

  /// With lazy materialization, functions which haven't been materialized when the module
  /// is finalized must be dead code or referenced only by global variables (e.g. boost
  /// library functions or virtual function (e.g. IfExpr::GetBooleanVal())), in which case
  /// the function is not inlined so the native version can be used and the IR version is
  /// dead code. Mark them as not materializable, change their linkage types to external
  /// (so linking will happen to the native version) and strip their personality functions
  /// and comdats. DCE may complain if the above are not done. Return error status if
  /// there is error in materializing the module.
  Status FinalizeLazyMaterialization();

  /// Destroy the IR module, freeing memory used by the IR. Any machine code that was
  /// generated is retained by the execution engine.
  void DestroyModule();

  /// Generate a string containing all jitted function names from fns_to_jit_compile_.
  /// The generation of the string is simply the concatenation of all the names by
  /// sequence in fns_to_jit_compile_. Would be used in the codegen cache lookup to
  /// confirm whether a cache entry matches the need of the LlvmCodeGen.
  std::string GetAllFunctionNames();

  /// Generate and store the hash code of all the function names. Will be used to
  /// codegen cache only.
  void GenerateFunctionNamesHashCode();

  /// Disable CPU attributes in 'cpu_attrs' that are not present in
  /// the '--llvm_cpu_attr_whitelist' flag. The same attributes in the input are
  /// always present in the output, except "+" is flipped to "-" for the disabled
  /// attributes. E.g. if 'cpu_attrs' is {"+x", "+y", "-z"} and the whitelist is
  /// {"x", "z"}, returns {"+x", "-y", "-z"}.
  static std::unordered_set<std::string> ApplyCpuAttrWhitelist(
      const std::unordered_set<std::string>& cpu_attrs);

  /// Whether InitializeLlvm() has been called.
  static bool llvm_initialized_;

  /// Host CPU name and attributes, filled in by InitializeLlvm().
  static std::string cpu_name_;
  /// The cpu_attrs_ should not be modified during the execution except for tests.
  static std::unordered_set<std::string> cpu_attrs_;

  /// Value of "target-features" attribute to be set on all IR functions. Derived from
  /// 'cpu_attrs_'. Using a consistent value for this attribute among
  /// hand-crafted IR and cross-compiled functions allow them to be inlined into each
  /// other.
  static std::string target_features_attr_;

  /// Mapping between CpuInfo flags and the corresponding strings.
  /// The key is mapped to the string as follows:
  /// CpuInfo flag -> enabled feature.
  /// Bitwise negation of CpuInfo flag -> disabled feature.
  const static std::map<int64_t, std::string> cpu_flag_mappings_;

  /// A global shared call graph for all IR functions in the main module.
  /// Used for determining dependencies when materializing IR functions.
  static CodegenCallGraph shared_call_graph_;

  /// Pointer to the FragmentState which owns this codegen object. Needed in
  /// InlineConstFnAttr() to access the query options.
  const FragmentState* state_;

  /// ID used for debugging (can be e.g. the fragment instance ID)
  std::string id_;

  /// Codegen counters
  RuntimeProfile* const profile_;

  /// MemTracker used for tracking memory consumed by codegen. Connected to a parent
  /// MemTracker if one was provided during initialization. Owned by the ObjectPool
  /// provided in the constructor.
  MemTracker* mem_tracker_;

  /// Time spent reading the .ir file from the file system.
  RuntimeProfile::Counter* load_module_timer_;

  /// Time spent creating the initial module with the cross-compiled Impala IR.
  RuntimeProfile::Counter* prepare_module_timer_;

  /// Time spent generating module bitcode.
  RuntimeProfile::Counter* module_bitcode_gen_timer_;

  /// Time spent for codegen cache look up and save.
  RuntimeProfile::Counter* codegen_cache_lookup_timer_;
  RuntimeProfile::Counter* codegen_cache_save_timer_;

  /// Time spent by ExecNodes while adding IR to the module. Update by
  /// FragmentInstanceState during its 'CODEGEN_START' state.
  RuntimeProfile::Counter* ir_generation_timer_;

  /// Time spent pruning unused functions.
  RuntimeProfile::Counter* function_prune_timer_;

  /// Time spent optimizing the module.
  RuntimeProfile::Counter* optimization_timer_;

  /// Time spent compiling the module.
  RuntimeProfile::Counter* compile_timer_;

  /// Total codegen time spent in the main thread.
  RuntimeProfile::Counter* main_thread_timer_;

  /// Total codegen time spent in the compiler (helper) thread.
  RuntimeProfile::ThreadCounters* compile_thread_counters_;

  /// Total size of bitcode modules loaded in bytes.
  RuntimeProfile::Counter* module_bitcode_size_;

  /// Number of functions and instructions that are optimized and compiled after pruning
  /// unused functions from the module.
  RuntimeProfile::Counter* num_functions_;
  RuntimeProfile::Counter* num_instructions_;

  /// Number of instructions after optimization.
  RuntimeProfile::Counter* num_opt_functions_;
  RuntimeProfile::Counter* num_opt_instructions_;

  /// Number of functions that are used and cached.
  RuntimeProfile::Counter* num_cached_functions_;

  /// Aggregated llvm thread counters. Also includes the phase represented by
  /// 'ir_generation_timer_' and hence is also updated by FragmentInstanceState.
  RuntimeProfile::ThreadCounters* llvm_thread_counters_;

  std::unique_ptr<Thread> async_compile_thread_;

  /// whether or not optimizations are enabled
  bool optimizations_enabled_;

  /// Whether or not codegen cache is enabled.
  bool codegen_cache_enabled_ = true;

  /// If true, the module is corrupt and we cannot codegen this query.
  /// TODO: we could consider just removing the offending function and attempting to
  /// codegen the rest of the query.  This requires more testing though to make sure
  /// that the error is recoverable.
  bool is_corrupt_;

  /// If true, the module has been compiled.  It is not valid to add additional
  /// functions after this point.
  bool is_compiled_;

  /// Error string that llvm will write to
  std::string error_string_;

  /// Top level llvm object.  Objects from different contexts do not share anything.
  /// We can have multiple instances of the LlvmCodeGen object in different threads
  std::unique_ptr<llvm::LLVMContext> context_;

  /// Top level codegen object. Contains everything to jit one 'unit' of code.  module_ is
  /// set by Init(). module_ is owned by the execution engine.
  llvm::Module* module_;

  // Execution/Jitting engine.
  std::unique_ptr<llvm::ExecutionEngine> execution_engine_;

  /// Object cache which is for the execution engine to write the compiled codegened
  /// functions to. Would be used as a part of CodeGen caching.
  std::shared_ptr<CodeGenObjectCache> engine_cache_;

  /// Object cache from the global CodeGen Cache, storing compiled codegened functions
  /// that align with the module of the current ExecutionEngine.
  /// Not null only when there is a cache hit.
  /// The purpose of it is to maintain the lifecycle of this CodeGenObjectCache in case
  /// it gets evicted from the global cache while in use.
  std::shared_ptr<CodeGenObjectCache> engine_cache_cached_;

  /// The memory manager used by 'execution_engine_'. Owned by 'execution_engine_'.
  ImpalaMCJITMemoryManager* memory_manager_;

  /// Functions parsed from pre-compiled module. Indexed by ImpalaIR::Function enum.
  std::vector<llvm::Function*> cross_compiled_functions_;

  /// Stores functions handcrafted by impala.  This does not contain cross compiled
  /// functions, only function that were generated from scratch at runtime. Does not
  /// overlap with loaded_functions_.
  std::vector<llvm::Function*> handcrafted_functions_;

  /// Stores the functions that have been finalized.
  std::unordered_set<llvm::Function*> finalized_functions_;

  /// A mapping of unique id to registered expr functions
  std::map<int64_t, llvm::Function*> registered_exprs_map_;

  /// A set of all the functions in 'registered_exprs_map_' for quick lookup.
  std::set<llvm::Function*> registered_exprs_;

  /// A cache of loaded llvm intrinsics
  std::map<llvm::Intrinsic::ID, llvm::Function*> llvm_intrinsics_;

  /// This is a cache of generated hash functions by byte size.  It is common
  /// for the caller to know the number of bytes to hash (e.g. tuple width) and
  /// we can codegen a loop unrolled hash function.
  std::map<int, llvm::Function*> hash_fns_;

  /// The locations of modules that have been linked. Uses hdfs file location as the key.
  /// Used to avoid linking the same module twice, which causes symbol collision errors.
  std::set<std::string> linked_modules_;

  /// Stores the functions to automatically JIT compile after FinalizeModule(). The
  /// 'CodegenFnPtrBase*' function pointers will be set to the functions compiled from the
  /// corresponding 'llvm::Function' objects.
  ///
  /// The functions are stored in a sorted map where the keys are the function names.
  /// This is because we need the function names in GetAllFunctionNames() (in sorted
  /// order) and in PruneModule() (in any order).
  ///
  /// There is a one-to-one correspondence between function names and 'llvm::Function'
  /// objects but an 'llvm::Function' object may correspond to multiple
  /// 'CodegenFnPtrBase*'s, for example if multiple 'SlotRef' expressions refer to the
  /// same slot and the 'llvm::Function' is reused. In these cases the function pointers
  /// corresponding to a single 'llvm::Function' are owned by different objects but they
  /// will be set to the same value.
  using LlvmFunctionWithFnPtrTargets =
      std::pair<llvm::Function*, std::vector<CodegenFnPtrBase*>>;
  std::map<llvm::StringRef, LlvmFunctionWithFnPtrTargets> fns_to_jit_compile_;

  /// The hash code generated from all the function names in fns_to_jit_compile_.
  /// Used by the codegen cache only.
  uint64_t function_names_hashcode_;

  /// The symbol emitted associated with 'execution_engine_'. Methods on
  /// llvm representation of a few common types.  Owned by context.
  llvm::PointerType* ptr_type_;             // int8_t*
  llvm::Type* void_type_;                   // void
  llvm::Type* string_value_type_;           // StringValue
  llvm::Type* timestamp_value_type_;        // TimestampValue
  llvm::Type* collection_value_type_;       // CollectionValue

  /// llvm constants to help with code gen verbosity
  llvm::Constant* true_value_;
  llvm::Constant* false_value_;

  /// The symbol emitter associated with 'execution_engine_'. Methods on
  /// 'symbol_emitter_' are called by 'execution_engine_' when code is emitted or
  /// freed. The lifetime of the symbol emitter must be longer than
  /// 'execution_engine_'.
  std::unique_ptr<CodegenSymbolEmitter> symbol_emitter_;

  /// Provides an implementation of a LLVM diagnostic handler and maintains the error
  /// information from its callbacks.
  class DiagnosticHandler {
   public:
    /// Returns the last error that was reported via DiagnosticHandlerFn() and then
    /// clears it. Returns an empty string otherwise. This should be called after any
    /// LLVM API call that can fail but returns error info via this mechanism.
    /// TODO: IMPALA-6038: use this to check and handle errors wherever needed.
    std::string GetErrorString();

    /// Handler function that sets the state on an instance of this class which is
    /// accessible via the LlvmCodeGen object passed to it using the 'context'
    /// input parameter.
    static void DiagnosticHandlerFn(const llvm::DiagnosticInfo &info, void *context);

   private:
    /// Contains the last error that was reported via DiagnosticHandlerFn().
    /// Is cleared by a call to GetErrorString().
    std::string error_str_;
  };

  DiagnosticHandler diagnostic_handler_;

  /// Very rough estimate of memory in bytes that the IR and the intermediate data
  /// structures used by the optimizer may consume per LLVM IR instruction to be
  /// optimized (after dead code is removed). The number is chosen to avoid pathological
  /// behaviour at either extreme: failing queries unnecessarily because the memory
  /// estimate is too high versus having large amounts of untracked memory because the
  /// estimate is too low.
  ///
  /// This was chosen by looking at the behaviour of TPC-H queries. Using the heap growth
  /// profile from gperftools reveal that LLVM allocated ~9mb of memory for fragments with
  /// ~17k total instructions in TPC-H Q2. Inspection of other TPC-H queries revealed
  /// that a typical fragment from a TPC-H query is < 5,000 instructions, which translates
  /// to 2.5MB, which is almost always lower than the runtime memory requirement of the
  /// fragment - so we are unlikely to fail queries unnecessarily.
  ///
  /// This assumes optimizer memory usage scales linearly with instruction count. This is
  /// true only if the size of functions is bounded, because some optimization passes
  /// (e.g. global value numbering) use time and memory that is super-linear in relation
  /// to the # of instructions in a function. So codegen should avoid generating
  /// arbitrarily large function.
  static constexpr int64_t ESTIMATED_OPTIMIZER_BYTES_PER_INST = 512;
};
}

#endif
```

// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements. See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership. The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License. You may obtain a copy of the License at
//
// http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied. See the License for the
// specific language governing permissions and limitations
// under the License.
#include "codegen/llvm-codegen.h"
#include "codegen/llvm-codegen-cache.h"
#include <fstream>
#include <mutex>
#include <sstream>
#include <unordered_set>
#include <boost/algorithm/string.hpp>
#include <boost/assert/source_location.hpp>
#include <gutil/strings/substitute.h>
#include <llvm/Bitcode/BitcodeReader.h>
#include <llvm/Bitcode/BitcodeWriter.h>
#include <llvm/ExecutionEngine/ExecutionEngine.h>
#include <llvm/ExecutionEngine/MCJIT.h>
#include <llvm/IR/Constants.h>
#include <llvm/IR/DataLayout.h>
#include <llvm/IR/DiagnosticInfo.h>
#include <llvm/IR/DiagnosticPrinter.h>
#include <llvm/IR/Function.h>
#include <llvm/IR/GlobalVariable.h>
#include <llvm/IR/InstIterator.h>
#include <llvm/IR/Verifier.h>
#include <llvm/Linker/Linker.h>
#include <llvm/Passes/PassBuilder.h>
#include <llvm/Support/CommandLine.h>
#include <llvm/Support/DynamicLibrary.h>
#include <llvm/Support/ErrorHandling.h>
#include <llvm/Support/Host.h>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Support/raw_ostream.h>
#include <llvm/Transforms/IPO/GlobalDCE.h>
#include <llvm/Transforms/IPO/Internalize.h>
#include <llvm/Transforms/Scalar.h>
#include <llvm/Transforms/Utils/BasicBlockUtils.h>
#include <llvm/Transforms/Utils/Cloning.h>
#include "codegen/codegen-anyval.h"
#include "codegen/codegen-callgraph.h"
#include "codegen/codegen-fn-ptr.h"
#include "codegen/codegen-symbol-emitter.h"
#include "codegen/impala-ir-data.h"
#include "codegen/instruction-counter.h"
#include "codegen/mcjit-mem-mgr.h"
#include "common/logging.h"
#include "exprs/anyval-util.h"
#include "gutil/sysinfo.h"
#include "impala-ir/impala-ir-names.h"
#include "runtime/collection-value.h"
#include "runtime/descriptors.h"
#include "runtime/exec-env.h"
#include "runtime/fragment-state.h"
#include "runtime/hdfs-fs-cache.h"
#include "runtime/lib-cache.h"
#include "runtime/mem-pool.h"
#include "runtime/mem-tracker.h"
#include "runtime/runtime-state.h"
#include "runtime/string-value.h"
#include "runtime/timestamp-value.h"
#include "util/cpu-info.h"
#include "util/debug-util.h"
#include "util/hdfs-util.h"
#include "util/path-builder.h"
#include "util/runtime-profile-counters.h"
#include "util/symbols-util.h"
#include "util/test-info.h"
#include "util/thread.h"
#include "util/thrift-debug-util.h"
#include "common/names.h"
using namespace strings;
using std::fstream;
using std::move;
DEFINE_bool(print_llvm_ir_instruction_count, false,
    "if true, prints the instruction counts of all JIT'd functions");
DEFINE_bool(disable_optimization_passes, false,
    "if true, disables llvm optimization passes (used for testing)");
DEFINE_bool(dump_ir, false, "if true, output IR after optimization passes");
DEFINE_bool(perf_map, false,
    "if true, generate /tmp/perf-<pid>.map file for linux perf symbols. "
    "This is not recommended for production use because it may affect performance.");
DEFINE_string(unopt_module_dir, "",
    "if set, saves unoptimized generated IR modules to the specified directory.");
DEFINE_string(opt_module_dir, "",
    "if set, saves optimized generated IR modules to the specified directory.");
DEFINE_string(asm_module_dir, "",
    "if set, saves disassembly for generated IR modules to the specified directory.");
DECLARE_string(local_library_dir);
// IMPALA-6291: AVX-512 and other CPU attrs the community doesn't routinely test are
// disabled. AVX-512 is affected by known bugs in LLVM 3.9.1. The following attrs that
// exist in LLVM 3.9.1 are disabled: avx512bw,avx512cd,avx512dq,avx512er,avx512f,
// avx512ifma,avx512pf,avx512vbmi,avx512vl,clflushopt,clwb,fma4,mwaitx.1.2,pcommit,pku,
// prefetchwt1,sgx,sha,sse4a,tbm,xop,xsavec,xsaves. If new attrs are added to LLVM,
// they will be disabled until added to this whitelist.
#ifdef __aarch64__
DEFINE_string_hidden(llvm_cpu_attr_whitelist, "crc,neon,fp-armv8,crypto",
    "(Experimental) a comma-separated list of LLVM CPU attribute flags that are enabled "
    "for runtime code generation. This flag is provided to enable additional LLVM CPU "
    "attribute flags for testing.");
#else
DEFINE_string_hidden(llvm_cpu_attr_whitelist, "adx,aes,avx,avx2,bmi,bmi2,cmov,cx16,f16c,"
    "fma,fsgsbase,hle,invpcid,lzcnt,mmx,movbe,pclmul,popcnt,prfchw,rdrnd,rdseed,rtm,smap,"
    "sse,sse2,sse3,sse4.1,sse4.2,ssse3,xsave,xsaveopt",
    "(Experimental) a comma-separated list of LLVM CPU attribute flags that are enabled "
    "for runtime code generation. The default flags are a known-good set that are "
    "routinely tested. This flag is provided to enable additional LLVM CPU attribute "
    "flags for testing.");
#endif
DEFINE_string_hidden(llvm_ir_opt, "Os",
    "The IR optimization level for pre-generated code; supports O1, O2, and Os.");
DECLARE_bool(enable_legacy_avx_support);
namespace impala {
const string LlvmCodeGen::ASYNC_CODEGEN_THREAD_COUNTERS_PREFIX = "CodegenCompileThread";
bool LlvmCodeGen::llvm_initialized_ = false;
string LlvmCodeGen::cpu_name_;
std::unordered_set<string> LlvmCodeGen::cpu_attrs_;
string LlvmCodeGen::target_features_attr_;
CodegenCallGraph LlvmCodeGen::shared_call_graph_;
const map<int64_t, std::string> LlvmCodeGen::cpu_flag_mappings_{
    {CpuInfo::SSSE3, "+ssse3"}, {CpuInfo::SSE4_1, "+sse4.1"},
    {CpuInfo::SSE4_2, "+sse4.2"}, {CpuInfo::POPCNT, "+popcnt"}, {CpuInfo::AVX, "+avx"},
    {CpuInfo::AVX2, "+avx2"}, {CpuInfo::PCLMULQDQ, "+pclmul"},
    {~(CpuInfo::SSSE3), "-ssse3"}, {~(CpuInfo::SSE4_1), "-sse4.1"},
    {~(CpuInfo::SSE4_2), "-sse4.2"}, {~(CpuInfo::POPCNT), "-popcnt"},
    {~(CpuInfo::AVX), "-avx"}, {~(CpuInfo::AVX2), "-avx2"},
    {~(CpuInfo::PCLMULQDQ), "-pclmul"}};
[[noreturn]] static void LlvmCodegenHandleError(
    void* user_data, const string& reason, bool gen_crash_diag) {
  LOG(FATAL) << "LLVM hit fatal error: " << reason.c_str();
}
Status LlvmCodeGen::InitializeLlvm(const char* procname, bool load_backend) {
  DCHECK(!llvm_initialized_);
  // Treat all functions as having the inline hint
  std::array<const char*, 2> argv = { { procname, "-inline-threshold=325" } };
  CHECK(llvm::cl::ParseCommandLineOptions(argv.size(), argv.data()));
  llvm::remove_fatal_error_handler();
  llvm::install_fatal_error_handler(LlvmCodegenHandleError);
  // These functions can *only* be called once per process and are used to set up
  // LLVM subsystems for code generation targeting the machine we're running on.
  llvm::InitializeNativeTarget();
  llvm::InitializeNativeTargetAsmPrinter();
  llvm::InitializeNativeTargetAsmParser();
  llvm::InitializeNativeTargetDisassembler();
  llvm_initialized_ = true;
  if (load_backend) {
    string path;
    // For test env, we have to load libfesupport.so to provide sym for LLVM.
    PathBuilder::GetFullBuildPath("service/libfesupport.so", &path);
    bool failed = llvm::sys::DynamicLibrary::LoadLibraryPermanently(path.c_str());
    DCHECK_EQ(failed, 0);
  }
  cpu_name_ = llvm::sys::getHostCPUName().str();
  LOG(INFO) << "CPU class for runtime code generation: " << cpu_name_;
  GetHostCPUAttrs(&cpu_attrs_);
  LOG(INFO) << "Detected CPU flags: " << boost::join(cpu_attrs_, ",");
  cpu_attrs_ = ApplyCpuAttrWhitelist(cpu_attrs_);
  target_features_attr_ = boost::join(cpu_attrs_, ",");
  LOG(INFO) << "CPU flags enabled for runtime code generation: " << target_features_attr_;
  // Write an empty map file for perf to find.
  if (FLAGS_perf_map) CodegenSymbolEmitter::WritePerfMap();
  ObjectPool init_pool;
  scoped_ptr<LlvmCodeGen> init_codegen;
  RETURN_IF_ERROR(LlvmCodeGen::CreateFromMemory(
      nullptr, &init_pool, nullptr, "init", &init_codegen));
  // LLVM will construct "use" lists only when the entire module is materialized.
  RETURN_IF_ERROR(init_codegen->MaterializeModule());
  // Validate the module by verifying that functions for all IRFunction::Type
  // can be found.
  for (int i = IRFunction::FN_START; i < IRFunction::FN_END; ++i) {
    DCHECK_EQ(FN_MAPPINGS[i].fn, i);
    const string& fn_name = FN_MAPPINGS[i].fn_name;
    if (init_codegen->module_->getFunction(fn_name) == nullptr) {
      const string& err_msg = Substitute("Failed to find function $0", fn_name);
      LOG(ERROR) << err_msg;
      return Status(err_msg);
    }
  }
  // Initialize the global shared call graph.
  shared_call_graph_.Init(init_codegen->module_);
  init_codegen->Close();
  return Status::OK();
}
LlvmCodeGen::LlvmCodeGen(FragmentState* state, ObjectPool* pool,
    MemTracker* parent_mem_tracker, const string& id)
  : state_(state),
    id_(id),
    profile_(RuntimeProfile::Create(pool, "CodeGen")),
    mem_tracker_(pool->Add(new MemTracker(profile_, -1, "CodeGen", parent_mem_tracker))),
    optimizations_enabled_(false),
    is_corrupt_(false),
    is_compiled_(false),
    context_(new llvm::LLVMContext()),
    module_(nullptr),
    memory_manager_(nullptr),
    cross_compiled_functions_(IRFunction::FN_END, nullptr) {
  DCHECK(llvm_initialized_) << "Must call LlvmCodeGen::InitializeLlvm first.";
  context_->setDiagnosticHandler(&DiagnosticHandler::DiagnosticHandlerFn, this);
  load_module_timer_ = ADD_TIMER(profile_, "LoadTime");
  prepare_module_timer_ = ADD_TIMER(profile_, "PrepareTime");
  codegen_cache_lookup_timer_ = ADD_TIMER(profile_, "CodegenCacheLookupTime");
  codegen_cache_save_timer_ = ADD_TIMER(profile_, "CodegenCacheSaveTime");
  module_bitcode_gen_timer_ = ADD_TIMER(profile_, "ModuleBitcodeGenTime");
  module_bitcode_size_ = ADD_COUNTER(profile_, "ModuleBitcodeSize", TUnit::BYTES);
  ir_generation_timer_ = ADD_TIMER(profile_, "IrGenerationTime");
  function_prune_timer_ = ADD_TIMER(profile_, "FunctionPruneTime");
  optimization_timer_ = ADD_TIMER(profile_, "OptimizationTime");
  compile_timer_ = ADD_TIMER(profile_, "CompileTime");
  main_thread_timer_ = ADD_TIMER(profile_, "MainThreadCodegenTime");
  compile_thread_counters_ = ADD_THREAD_COUNTERS(profile_,
      ASYNC_CODEGEN_THREAD_COUNTERS_PREFIX);
  num_functions_ = ADD_COUNTER(profile_, "NumFunctions", TUnit::UNIT);
  num_instructions_ = ADD_COUNTER(profile_, "NumInstructions", TUnit::UNIT);
  num_opt_functions_ = ADD_COUNTER(profile_, "NumOptimizedFunctions", TUnit::UNIT);
  num_opt_instructions_ = ADD_COUNTER(profile_, "NumOptimizedInstructions", TUnit::UNIT);
  num_cached_functions_ = ADD_COUNTER(profile_, "NumCachedFunctions", TUnit::UNIT);
  llvm_thread_counters_ = ADD_THREAD_COUNTERS(profile_, "Codegen");
}
Status LlvmCodeGen::CreateFromFile(FragmentState* state, ObjectPool* pool,
    MemTracker* parent_mem_tracker, const string& file, const string& id,
    scoped_ptr<LlvmCodeGen>* codegen) {
  codegen->reset(new LlvmCodeGen(state, pool, parent_mem_tracker, id));
  SCOPED_TIMER((*codegen)->profile_->total_time_counter());
  SCOPED_THREAD_COUNTER_MEASUREMENT((*codegen)->llvm_thread_counters());
  unique_ptr<llvm::Module> loaded_module;
  Status status = (*codegen)->LoadModuleFromFile(file, &loaded_module);
  if (!status.ok()) goto error;
  status = (*codegen)->Init(move(loaded_module));
  if (!status.ok()) goto error;
  return Status::OK();
error:
  (*codegen)->Close();
  return status;
}
Status LlvmCodeGen::CreateFromMemory(FragmentState* state, ObjectPool* pool,
    MemTracker* parent_mem_tracker, const string& id, scoped_ptr<LlvmCodeGen>* codegen) {
  codegen->reset(new LlvmCodeGen(state, pool, parent_mem_tracker, id));
  SCOPED_TIMER((*codegen)->profile_->total_time_counter());
  SCOPED_TIMER((*codegen)->prepare_module_timer_);
  SCOPED_THREAD_COUNTER_MEASUREMENT((*codegen)->llvm_thread_counters());
  llvm::StringRef module_ir;
  string module_name = "Impala IR";
  if (FLAGS_llvm_ir_opt == "O1") {
    module_ir = llvm::StringRef(
        reinterpret_cast<const char*>(impala_llvm_o1_ir), impala_llvm_o1_ir_len);
  } else if (FLAGS_llvm_ir_opt == "O2") {
    module_ir = llvm::StringRef(
        reinterpret_cast<const char*>(impala_llvm_o2_ir), impala_llvm_o2_ir_len);
  } else if (FLAGS_llvm_ir_opt == "Os") {
    module_ir = llvm::StringRef(
        reinterpret_cast<const char*>(impala_llvm_os_ir), impala_llvm_os_ir_len);
  } else {
    CHECK(false) << "llvm_ir_opt flag invalid; try O1, O2, or Os.";
  }
#if __x86_64__
  // By default, Impala now requires AVX2 support, but the enable_legacy_avx_support
  // flag can allow running on AVX machines. The minimum requirement must have already
  // been enforced prior to this call, so this only needs to select the appropriate
  // LLVM IR to use.
  if (IsCPUFeatureEnabled(CpuInfo::AVX2)) {
    // Use the default IR that supports AVX2
    module_name = "Impala IR with AVX2 support";
  } else if (FLAGS_enable_legacy_avx_support && IsCPUFeatureEnabled(CpuInfo::AVX)) {
    // If there is no AVX but legacy mode is enabled, use legacy IR with AVX support
    module_ir = llvm::StringRef(
        reinterpret_cast<const char*>(impala_legacy_avx_llvm_ir),
        impala_legacy_avx_llvm_ir_len);
    module_name = "Legacy Impala IR with AVX support";
  } else {
    // This should have been enforced earlier.
    CHECK(false) << "CPU is missing AVX/AVX2 support";
  }
#endif
  unique_ptr<llvm::MemoryBuffer> module_ir_buf(
      llvm::MemoryBuffer::getMemBuffer(module_ir, "", false));
  unique_ptr<llvm::Module> loaded_module;
  Status status = (*codegen)->LoadModuleFromMemory(move(module_ir_buf),
      module_name, &loaded_module);
  if (!status.ok()) goto error;
  status = (*codegen)->Init(move(loaded_module));
  if (!status.ok()) goto error;
  return Status::OK();
error:
  (*codegen)->Close();
  return status;
}
Status LlvmCodeGen::LoadModuleFromFile(
    const string& file, unique_ptr<llvm::Module>* module) {
  unique_ptr<llvm::MemoryBuffer> file_buffer;
  {
    SCOPED_TIMER(load_module_timer_);
    llvm::ErrorOr<unique_ptr<llvm::MemoryBuffer>> tmp_file_buffer =
        llvm::MemoryBuffer::getFile(file);
    if (!tmp_file_buffer) {
      stringstream ss;
      ss << "Could not load module " << file << ": "
         << tmp_file_buffer.getError().message();
      return Status(ss.str());
    }
    file_buffer = move(tmp_file_buffer.get());
  }
  COUNTER_ADD(module_bitcode_size_, file_buffer->getBufferSize());
  return LoadModuleFromMemory(move(file_buffer), file, module);
}
Status LlvmCodeGen::LoadModuleFromMemory(unique_ptr<llvm::MemoryBuffer> module_ir_buf,
    const string& module_name, unique_ptr<llvm::Module>* module) {
  DCHECK(!module_name.empty());
  COUNTER_ADD(module_bitcode_size_, module_ir_buf->getMemBufferRef().getBufferSize());
  llvm::Expected<unique_ptr<llvm::Module>> tmp_module =
      getOwningLazyBitcodeModule(move(module_ir_buf), context());
  if (llvm::Error err = tmp_module.takeError()) {
    string err_string;
    llvm::handleAllErrors(
        move(err), [&](llvm::ErrorInfoBase& eib) { err_string = eib.message(); });
    return Status(err_string);
  }
  *module = move(tmp_module.get());
  // We never run global constructors or destructors so let's strip them out for all
  // modules when we load them.
  StripGlobalCtorsDtors((*module).get());
  (*module)->setModuleIdentifier(module_name);
  return Status::OK();
}
// TODO: Create separate counters/timers (file size, load time) for each module linked
Status LlvmCodeGen::LinkModuleFromLocalFs(const string& file) {
  unique_ptr<llvm::Module> new_module;
  RETURN_IF_ERROR(LoadModuleFromFile(file, &new_module));
  // The module data layout must match the one selected by the execution engine.
  new_module->setDataLayout(execution_engine()->getDataLayout());
  // Parse all functions' names from the new module and find those which also exist in
  // the main module. They are declarations in the new module or duplicated definitions
  // of the same function in both modules. For the latter case, it's unclear which one
  // the linker will choose. Materialize these functions in the main module in case they
  // are chosen by the linker or referenced by functions in the new module. Note that
  // linkModules() will materialize functions defined only in the new module.
  for (llvm::Function& fn : new_module->functions()) {
    const string& fn_name = fn.getName();
    if (shared_call_graph_.GetCallees(fn_name) != nullptr) {
      llvm::Function* local_fn = module_->getFunction(fn_name);
      RETURN_IF_ERROR(MaterializeFunction(local_fn));
    }
  }
  bool error = llvm::Linker::linkModules(*module_, move(new_module));
  string diagnostic_err = diagnostic_handler_.GetErrorString();
  if (error) {
    stringstream ss;
    ss << "Problem linking " << file << " to main module.";
    if (!diagnostic_err.empty()) ss << " " << diagnostic_err;
    return Status(ss.str());
  }
  return Status::OK();
}
Status LlvmCodeGen::LinkModuleFromHdfs(const string& hdfs_location, const time_t mtime) {
  if (linked_modules_.find(hdfs_location) != linked_modules_.end()) return Status::OK();
  LibCacheEntryHandle handle;
  string local_path;
  RETURN_IF_ERROR(LibCache::instance()->GetLocalPath(
      hdfs_location, LibCache::TYPE_IR, mtime, &handle, &local_path));
  RETURN_IF_ERROR(LinkModuleFromLocalFs(local_path));
  linked_modules_.insert(hdfs_location);
  return Status::OK();
}
void LlvmCodeGen::StripGlobalCtorsDtors(llvm::Module* module) {
  llvm::GlobalVariable* constructors = module->getGlobalVariable("llvm.global_ctors");
  if (constructors != NULL) constructors->eraseFromParent();
  llvm::GlobalVariable* destructors = module->getGlobalVariable("llvm.global_dtors");
  if (destructors != NULL) destructors->eraseFromParent();
}
Status LlvmCodeGen::CreateImpalaCodegen(FragmentState* state,
    MemTracker* parent_mem_tracker, const string& id,
    scoped_ptr<LlvmCodeGen>* codegen_ret) {
  DCHECK(state != nullptr);
  RETURN_IF_ERROR(CreateFromMemory(
      state, state->obj_pool(), parent_mem_tracker, id, codegen_ret));
  LlvmCodeGen* codegen = codegen_ret->get();
  // Parse module for cross compiled functions and types
  SCOPED_TIMER(codegen->profile_->total_time_counter());
  SCOPED_TIMER(codegen->prepare_module_timer_);
  SCOPED_THREAD_COUNTER_MEASUREMENT(codegen->llvm_thread_counters_);
  // Get type for StringValue
  codegen->string_value_type_ = codegen->GetStructType<StringValue>();
  // Get type for TimestampValue
  codegen->timestamp_value_type_ = codegen->GetStructType<TimestampValue>();
  // Get type for CollectionValue
  codegen->collection_value_type_ = codegen->GetStructType<CollectionValue>();
  // Verify size is correct
  const llvm::DataLayout& data_layout = codegen->execution_engine()->getDataLayout();
  const llvm::StructLayout* layout = data_layout.getStructLayout(
      static_cast<llvm::StructType*>(codegen->string_value_type_));
  if (layout->getSizeInBytes() != sizeof(StringValue)) {
    DCHECK_EQ(layout->getSizeInBytes(), sizeof(StringValue));
    return Status("Could not create llvm struct type for StringVal");
  }
  // Materialize functions referenced by the global variables.
  for (const string& fn_name : shared_call_graph_.fns_referenced_by_gv()) {
    llvm::Function* fn = codegen->module_->getFunction(fn_name);
    DCHECK(fn != nullptr);
    RETURN_IF_ERROR(codegen->MaterializeFunction(fn));
  }
  return Status::OK();
}
Status LlvmCodeGen::Init(unique_ptr<llvm::Module> module) {
  DCHECK(module != nullptr);
  llvm::CodeGenOpt::Level opt_level = llvm::CodeGenOpt::Aggressive;
#ifndef NDEBUG
  // For debug builds, don't generate JIT compiled optimized assembly.
  // This takes a non-neglible amount of time (~.5 ms per function) and
  // blows up the fe tests (which take ~10-20 ms each).
  opt_level = llvm::CodeGenOpt::None;
#endif
  module_ = module.get();
  llvm::EngineBuilder builder(move(module));
  builder.setEngineKind(llvm::EngineKind::JIT);
  builder.setOptLevel(opt_level);
  unique_ptr<ImpalaMCJITMemoryManager> memory_manager(new ImpalaMCJITMemoryManager);
  memory_manager_ = memory_manager.get();
  builder.setMCJITMemoryManager(move(memory_manager));
  builder.setMCPU(cpu_name_);
  builder.setMAttrs(cpu_attrs_);
  builder.setErrorStr(&error_string_);
  execution_engine_ = unique_ptr<llvm::ExecutionEngine>(builder.create());
  if (execution_engine_ == nullptr) {
    module_ = nullptr; // module_ was owned by builder.
    stringstream ss;
    ss << "Could not create ExecutionEngine: " << error_string_;
    return Status(ss.str());
  }
  // The module data layout must match the one selected by the execution engine.
  module_->setDataLayout(execution_engine_->getDataLayout());
  void_type_ = llvm::Type::getVoidTy(context());
  ptr_type_ = llvm::PointerType::get(i8_type(), 0);
  true_value_ = llvm::ConstantInt::get(context(), llvm::APInt(1, true, true));
  false_value_ = llvm::ConstantInt::get(context(), llvm::APInt(1, false, true));
  symbol_emitter_ = SetupSymbolEmitter(execution_engine_.get());
  engine_cache_ = make_shared<CodeGenObjectCache>();
  RETURN_IF_ERROR(LoadIntrinsics());
  return Status::OK();
}
unique_ptr<CodegenSymbolEmitter> LlvmCodeGen::SetupSymbolEmitter(
    llvm::ExecutionEngine* execution_engine) {
  bool need_symbol_emitter = !FLAGS_asm_module_dir.empty() || FLAGS_perf_map;
  if (!need_symbol_emitter) return nullptr;
  unique_ptr<CodegenSymbolEmitter> symbol_emitter =
      make_unique<CodegenSymbolEmitter>(id_);
  execution_engine->RegisterJITEventListener(symbol_emitter.get());
  symbol_emitter->set_emit_perf_map(FLAGS_perf_map);
  if (!FLAGS_asm_module_dir.empty()) {
    symbol_emitter->set_asm_path(Substitute("$0/$1.asm", FLAGS_asm_module_dir, id_));
  }
  return symbol_emitter;
}
LlvmCodeGen::~LlvmCodeGen() {
  DCHECK(execution_engine_ == nullptr) << "Must Close() before destruction";
}
void LlvmCodeGen::Close() {
  if (async_compile_thread_ != nullptr) async_compile_thread_->Join();
  if (memory_manager_ != nullptr) {
    mem_tracker_->Release(memory_manager_->bytes_tracked());
    memory_manager_ = nullptr;
  }
  if (mem_tracker_ != nullptr) mem_tracker_->Close();
  engine_cache_.reset();
  engine_cache_cached_.reset();
  execution_engine_.reset();
  symbol_emitter_.reset();
  module_ = nullptr;
}
void LlvmCodeGen::EnableOptimizations(bool enable) {
  optimizations_enabled_ = enable;
}
void LlvmCodeGen::GetHostCPUAttrs(std::unordered_set<string>* attrs) {
  // LLVM's ExecutionEngine expects features to be enabled or disabled with a list
  // of strings like ["+feature1", "-feature2"].
  llvm::StringMap<bool> cpu_features;
  llvm::sys::getHostCPUFeatures(cpu_features);
  for (const llvm::StringMapEntry<bool>& entry : cpu_features) {
    attrs->emplace(Substitute("$0$1", entry.second ? "+" : "-", entry.first().data()));
  }
}
bool LlvmCodeGen::IsCPUFeatureEnabled(int64_t flag) {
  DCHECK(llvm_initialized_);
  auto enable_flag_it = cpu_flag_mappings_.find(flag);
  DCHECK(enable_flag_it != cpu_flag_mappings_.end());
  const std::string& enabled_feature = enable_flag_it->second;
  return cpu_attrs_.find(enabled_feature) != cpu_attrs_.end();
}
string LlvmCodeGen::GetIR(bool full_module) const {
  string str;
  llvm::raw_string_ostream stream(str);
  if (full_module) {
    module_->print(stream, NULL);
  } else {
    for (int i = 0; i < handcrafted_functions_.size(); ++i) {
      handcrafted_functions_[i]->print(stream, nullptr, false, true);
    }
  }
  return str;
}
llvm::Type* LlvmCodeGen::GetSlotType(const ColumnType& type) {
  switch (type.type) {
    case TYPE_NULL:
      return llvm::Type::getInt1Ty(context());
    case TYPE_BOOLEAN:
      return bool_type();
    case TYPE_TINYINT:
      return i8_type();
    case TYPE_SMALLINT:
      return i16_type();
    case TYPE_INT:
      return i32_type();
    case TYPE_BIGINT:
      return i64_type();
    case TYPE_FLOAT:
      return float_type();
    case TYPE_DOUBLE:
      return double_type();
    case TYPE_STRING:
    case TYPE_VARCHAR:
      return string_value_type_;
    case TYPE_CHAR:
    case TYPE_FIXED_UDA_INTERMEDIATE:
      // Represent this as an array of bytes.
      return llvm::ArrayType::get(i8_type(), type.len);
    case TYPE_TIMESTAMP:
      return timestamp_value_type_;
    case TYPE_DECIMAL:
      return llvm::Type::getIntNTy(context(), type.GetByteSize() * 8);
    case TYPE_DATE:
      return i32_type();
    case TYPE_ARRAY:
    case TYPE_MAP:
      return collection_value_type_;
    default:
      DCHECK(false) << "Invalid type: " << type;
      return NULL;
  }
}
llvm::PointerType* LlvmCodeGen::GetSlotPtrType(const ColumnType& type) {
  return llvm::PointerType::get(GetSlotType(type), 0);
}
llvm::Type* LlvmCodeGen::GetNamedType(const string& name) {
  llvm::Type* type = module_->getTypeByName(name);
  DCHECK(type != NULL) << name;
  return type;
}
llvm::PointerType* LlvmCodeGen::GetNamedPtrType(const string& name) {
  llvm::Type* type = GetNamedType(name);
  DCHECK(type != NULL) << name;
  return llvm::PointerType::get(type, 0);
}
llvm::PointerType* LlvmCodeGen::GetPtrType(llvm::Type* type) {
  return llvm::PointerType::get(type, 0);
}
llvm::PointerType* LlvmCodeGen::GetPtrPtrType(llvm::Type* type) {
  return llvm::PointerType::get(llvm::PointerType::get(type, 0), 0);
}
llvm::PointerType* LlvmCodeGen::GetNamedPtrPtrType(const string& name) {
  return llvm::PointerType::get(GetNamedPtrType(name), 0);
}
llvm::Constant* LlvmCodeGen::GetIntConstant(
    int num_bytes, uint64_t low_bits, uint64_t high_bits) {
  DCHECK_GE(num_bytes, 1);
  DCHECK_LE(num_bytes, 16);
  DCHECK(BitUtil::IsPowerOf2(num_bytes));
  vector<uint64_t> vals({low_bits, high_bits});
  return llvm::ConstantInt::get(context(), llvm::APInt(8 * num_bytes, vals));
}
llvm::Value* LlvmCodeGen::GetStringConstant(
    LlvmBuilder* builder, const char* data, int len) {
  // Create a global string with private linkage.
  llvm::Constant* const_string =
      llvm::ConstantDataArray::getString(context(), llvm::StringRef(data, len), false);
  llvm::GlobalVariable* gv = new llvm::GlobalVariable(*module_, const_string->getType(),
      true, llvm::GlobalValue::PrivateLinkage, const_string);
  // Get a pointer to the first element of the string.
  return builder->CreateConstInBoundsGEP2_32(NULL, gv, 0, 0, "");
}
llvm::AllocaInst* LlvmCodeGen::CreateEntryBlockAlloca(
    llvm::Function* f, const NamedVariable& var) {
  llvm::IRBuilder<> tmp(&f->getEntryBlock(), f->getEntryBlock().begin());
  llvm::AllocaInst* alloca = tmp.CreateAlloca(var.type, NULL, var.name.c_str());
  if (var.type == GetNamedType(CodegenAnyVal::LLVM_DECIMALVAL_NAME)) {
    // Generated functions may manipulate DecimalVal arguments via SIMD instructions such
    // as 'movaps' that require 16-byte memory alignment. LLVM uses 8-byte alignment by
    // default, so explicitly set the alignment for DecimalVals.
    alloca->setAlignment(16);
  }
  return alloca;
}
llvm::AllocaInst* LlvmCodeGen::CreateEntryBlockAlloca(
    const LlvmBuilder& builder, llvm::Type* type, const char* name) {
  return CreateEntryBlockAlloca(
      builder.GetInsertBlock()->getParent(), NamedVariable(name, type));
}
llvm::AllocaInst* LlvmCodeGen::CreateEntryBlockAlloca(const LlvmBuilder& builder,
    llvm::Type* type, int num_entries, int alignment, const char* name) {
  llvm::Function* fn = builder.GetInsertBlock()->getParent();
  llvm::IRBuilder<> tmp(&fn->getEntryBlock(), fn->getEntryBlock().begin());
  llvm::AllocaInst* alloca =
      tmp.CreateAlloca(type, GetI32Constant(num_entries), name);
  alloca->setAlignment(alignment);
  return alloca;
}
void LlvmCodeGen::CreateIfElseBlocks(llvm::Function* fn, const string& if_name,
    const string& else_name, llvm::BasicBlock** if_block, llvm::BasicBlock** else_block,
    llvm::BasicBlock* insert_before) {
  *if_block = llvm::BasicBlock::Create(context(), if_name, fn, insert_before);
  *else_block = llvm::BasicBlock::Create(context(), else_name, fn, insert_before);
}
llvm::PHINode* LlvmCodeGen::CreateBinaryPhiNode(LlvmBuilder* builder, llvm::Value* value1,
    llvm::Value* value2, llvm::BasicBlock* incoming_block1,
    llvm::BasicBlock* incoming_block2, const string& name) {
  llvm::PHINode* res = builder->CreatePHI(value1->getType(), 2, name);
  res->addIncoming(value1, incoming_block1);
  res->addIncoming(value2, incoming_block2);
  return res;
}
Status LlvmCodeGen::MaterializeFunction(llvm::Function* fn) {
  DCHECK(!is_compiled_);
  if (fn->isIntrinsic() || !fn->isMaterializable()) return Status::OK();
  llvm::Error err = module_->materialize(fn);
  if (UNLIKELY(err)) {
    string err_string;
    llvm::handleAllErrors(
        move(err), [&](llvm::ErrorInfoBase& eib) { err_string = eib.message(); });
    return Status(Substitute("Failed to materialize $0: $1",
        fn->getName().str(), err_string));
  }
  // Materialized functions are marked as not materializable by LLVM.
  DCHECK(!fn->isMaterializable());
  SetCPUAttrs(fn);
  const unordered_set<string>* callees = shared_call_graph_.GetCallees(fn->getName());
  if (callees != nullptr) {
    for (const string& callee : *callees) {
      llvm::Function* callee_fn = module_->getFunction(callee);
      DCHECK(callee_fn != nullptr);
      RETURN_IF_ERROR(MaterializeFunction(callee_fn));
    }
  }
  return Status::OK();
}
llvm::Function* LlvmCodeGen::GetFunction(const string& symbol, bool clone) {
  llvm::Function* fn = module_->getFunction(symbol.c_str());
  if (fn == NULL) {
    LOG(ERROR) << "Unable to locate function " << symbol;
    return NULL;
  }
  Status status = MaterializeFunction(fn);
  if (UNLIKELY(!status.ok())) return NULL;
  if (clone) return CloneFunction(fn);
  return fn;
}
llvm::Function* LlvmCodeGen::GetFunction(IRFunction::Type ir_type, bool clone) {
  llvm::Function* fn = cross_compiled_functions_[ir_type];
  if (fn == NULL) {
    DCHECK_EQ(FN_MAPPINGS[ir_type].fn, ir_type);
    const string& fn_name = FN_MAPPINGS[ir_type].fn_name;
    fn = module_->getFunction(fn_name);
    if (fn == NULL) {
      LOG(ERROR) << "Unable to locate function " << fn_name;
      return NULL;
    }
    cross_compiled_functions_[ir_type] = fn;
  }
  Status status = MaterializeFunction(fn);
  if (UNLIKELY(!status.ok())) return NULL;
  if (clone) return CloneFunction(fn);
  return fn;
}
// TODO: this should return a Status
bool LlvmCodeGen::VerifyFunction(llvm::Function* fn) {
  if (is_corrupt_) return false;
  // Check that there are no calls to FunctionContextImpl::GetConstFnAttr(). These should all
  // have been inlined via InlineConstFnAttrs().
  for (llvm::inst_iterator iter = inst_begin(fn); iter != inst_end(fn); ++iter) {
    llvm::Instruction* instr = &*iter;
    if (!llvm::isa<llvm::CallInst>(instr)) continue;
    llvm::CallInst* call_instr = reinterpret_cast<llvm::CallInst*>(instr);
    llvm::Function* called_fn = call_instr->getCalledFunction();
    // Look for call to FunctionContextImpl::GetConstFnAttr().
    if (called_fn != nullptr &&
        called_fn->getName() == FunctionContextImpl::GET_CONST_FN_ATTR_SYMBOL) {
      LOG(ERROR) << "Found call to FunctionContextImpl::GetConstFnAttr(): "
                 << Print(call_instr);
      is_corrupt_ = true;
      break;
    }
  }
  // There is an llvm bug (#10957) that causes the first step of the verifier to always
  // abort the process if it runs into an issue and ignores ReturnStatusAction. This
  // would cause impalad to go down if one query has a problem. To work around this, we
  // will copy that step here and not abort on error. Adapted from the pre-verifier
  // function pass.
  // TODO: doesn't seem there is much traction in getting this fixed but we'll see
  for (llvm::Function::iterator i = fn->begin(), e = fn->end(); i != e; ++i) {
    if (i->empty() || !i->back().isTerminator()) {
      LOG(ERROR) << "Basic block must end with terminator: \n" << Print(&(*i));
      is_corrupt_ = true;
      break;
    }
  }
  if (!is_corrupt_) {
    string str;
    llvm::raw_string_ostream stream(str);
    is_corrupt_ = verifyFunction(*fn, &stream);
    if (is_corrupt_) LOG(ERROR) << str;
  }
  if (is_corrupt_) {
    string fn_name = fn->getName(); // llvm has some fancy operator overloading
    LOG(ERROR) << "Function corrupt: " << fn_name <<"\nFunction Dump: "
        << LlvmCodeGen::Print(fn);
    return false;
  }
  return true;
}
void LlvmCodeGen::SetNoInline(llvm::Function* function) const {
  function->removeFnAttr(llvm::Attribute::AlwaysInline);
  function->removeFnAttr(llvm::Attribute::InlineHint);
  function->addFnAttr(llvm::Attribute::NoInline);
}
void LlvmCodeGen::SetCPUAttrs(llvm::Function* function) {
  // Set all functions' "target-cpu" and "target-features" to match the
  // host's CPU's features. It's assumed that the functions don't use CPU
  // features which the host doesn't support. CreateFromMemory() checks
  // the features of the host's CPU and loads the module compatible with
  // the host's CPU.
  function->addFnAttr("target-cpu", cpu_name_);
  function->addFnAttr("target-features", target_features_attr_);
}
LlvmCodeGen::FnPrototype::FnPrototype(
    LlvmCodeGen* codegen, const string& name, llvm::Type* ret_type)
  : codegen_(codegen), name_(name), ret_type_(ret_type) {
  DCHECK(!codegen_->is_compiled_) << "Not valid to add additional functions";
}
llvm::Function* LlvmCodeGen::FnPrototype::GeneratePrototype(
    LlvmBuilder* builder, llvm::Value** params) {
  vector<llvm::Type*> arguments;
  arguments.reserve(args_.size());
  for (int i = 0; i < args_.size(); ++i) {
    arguments.push_back(args_[i].type);
  }
  llvm::FunctionType* prototype = llvm::FunctionType::get(ret_type_, arguments, false);
  llvm::Function* fn = llvm::Function::Create(
      prototype, llvm::GlobalValue::ExternalLinkage, name_, codegen_->module_);
  DCHECK(fn != NULL);
  // Name the arguments
  int idx = 0;
  for (llvm::Function::arg_iterator iter = fn->arg_begin(); iter != fn->arg_end();
       ++iter, ++idx) {
    iter->setName(args_[idx].name);
    if (params != NULL) params[idx] = &*iter;
  }
  if (builder != NULL) {
    llvm::BasicBlock* entry_block =
        llvm::BasicBlock::Create(codegen_->context(), "entry", fn);
    // Add it to the llvm module via the builder and to the list of handcrafted functions
    // that are a part of the module.
    builder->SetInsertPoint(entry_block);
    codegen_->handcrafted_functions_.push_back(fn);
  }
  return fn;
}
Status LlvmCodeGen::LoadFunction(const TFunction& fn, const string& symbol,
    const ColumnType* return_type, const vector<ColumnType>& arg_types,
    int num_fixed_args, bool has_varargs, llvm::Function** llvm_fn,
    LibCacheEntry** cache_entry) {
  DCHECK_GE(arg_types.size(), num_fixed_args);
  DCHECK(has_varargs || arg_types.size() == num_fixed_args);
  DCHECK(!has_varargs || arg_types.size() > num_fixed_args);
  // from_utc_timestamp() and to_utc_timestamp() have inline ASM that cannot be JIT'd.
  // TimestampFunctions::AddSub() contains a try/catch which doesn't work in JIT'd
  // code. Always use the interpreted version of these functions.
  // TODO: fix these built-in functions so we don't need 'broken_builtin' below.
  bool broken_builtin = fn.name.function_name == "from_utc_timestamp"
      || fn.name.function_name == "to_utc_timestamp"
      || symbol.find("AddSub") != string::npos;
  if (fn.binary_type == TFunctionBinaryType::NATIVE
      || (fn.binary_type == TFunctionBinaryType::BUILTIN && broken_builtin)) {
    // In this path, we are calling a precompiled native function, either a UDF
    // in a .so or a builtin using the UDF interface.
    void* fn_ptr;
    Status status = LibCache::instance()->GetSoFunctionPtr(
        fn.hdfs_location, symbol, fn.last_modified_time, &fn_ptr, cache_entry);
    if (!status.ok() && fn.binary_type == TFunctionBinaryType::BUILTIN) {
      // Builtins symbols should exist unless there is a version mismatch.
      status.AddDetail(
          ErrorMsg(TErrorCode::MISSING_BUILTIN, fn.name.function_name, symbol).msg());
    }
    RETURN_IF_ERROR(status);
    DCHECK(fn_ptr != NULL);
    // Per the x64 ABI, DecimalVals are returned via a DecimalVal* output argument.
    // So, the return type is void.
    bool is_decimal = return_type != NULL && return_type->type == TYPE_DECIMAL;
    llvm::Type* llvm_return_type = return_type == NULL || is_decimal ?
        void_type() :
        CodegenAnyVal::GetLoweredType(this, *return_type);
    // Convert UDF function pointer to Function*. Start by creating a function
    // prototype for it.
    FnPrototype prototype(this, symbol, llvm_return_type);
    if (is_decimal) {
      // Per the x64 ABI, DecimalVals are returned via a DecmialVal* output argument
      llvm::Type* output_type = CodegenAnyVal::GetUnloweredPtrType(this, *return_type);
      prototype.AddArgument("output", output_type);
    }
    // The "FunctionContext*" argument.
    prototype.AddArgument("ctx", GetNamedPtrType("class.impala_udf::FunctionContext"));
    // The "fixed" arguments for the UDF function, followed by the variable arguments,
    // if any.
    for (int i = 0; i < num_fixed_args; ++i) {
      llvm::Type* arg_type = CodegenAnyVal::GetUnloweredPtrType(this, arg_types[i]);
      prototype.AddArgument(Substitute("fixed_arg_$0", i), arg_type);
    }
    if (has_varargs) {
      prototype.AddArgument("num_var_arg", i32_type());
      // Get the vararg type from the first vararg.
      prototype.AddArgument(
          "var_arg", CodegenAnyVal::GetUnloweredPtrType(this, arg_types[num_fixed_args]));
    }
    // Create a Function* with the generated type. This is only a function
    // declaration, not a definition, since we do not create any basic blocks or
    // instructions in it.
    *llvm_fn = prototype.GeneratePrototype(nullptr, nullptr);
#ifdef __aarch64__
    if (is_decimal) {
      // Mark first argument as sret
      (*llvm_fn)->addAttribute(1, llvm::Attribute::StructRet);
    }
#endif
    // Associate the dynamically loaded function pointer with the Function* we defined.
    // This tells LLVM where the compiled function definition is located in memory.
    execution_engine()->addGlobalMapping(*llvm_fn, fn_ptr);
    // Disable the codegen cache because codegen cache uses the llvm module bitcode as
    // the key while the bitcode doesn't contain the global function mapping of the
    // execution engine. If the mapping is changed during running, like udf recreation,
    // the function mapping in the cache could point to an old address and lead to a
    // crash while calling the udf, so block the codegen cache for native udfs.
    // Builtin functions should not have the issue, because they should not change
    // during runtime.
    if (fn.binary_type == TFunctionBinaryType::NATIVE) {
      // Should be before compilation.
      DCHECK(!is_compiled_);
      codegen_cache_enabled_ = false;
    }
  } else if (fn.binary_type == TFunctionBinaryType::BUILTIN) {
    // In this path, we're running a builtin with the UDF interface. The IR is
    // in the llvm module. Builtin functions may use Expr::GetConstant(). Clone the
    // function so that we can replace constants in the copied function.
    *llvm_fn = GetFunction(symbol, true);
    if (*llvm_fn == NULL) {
      // Builtins symbols should exist unless there is a version mismatch.
      return Status(Substitute("Builtin '$0' with symbol '$1' does not exist. Verify "
                               "that all your impalads are the same version.",
          fn.name.function_name, symbol));
    }
    // Rename the function to something more readable than the mangled name.
    string demangled_name = SymbolsUtil::DemangleNoArgs((*llvm_fn)->getName().str());
    (*llvm_fn)->setName(demangled_name);
  } else {
    // We're running an IR UDF.
    DCHECK_EQ(fn.binary_type, TFunctionBinaryType::IR);
    // Link the UDF module into this query's main module so the UDF's functions are
    // available in the main module.
    RETURN_IF_ERROR(LinkModuleFromHdfs(fn.hdfs_location, fn.last_modified_time));
    *llvm_fn = GetFunction(symbol, true);
    if (*llvm_fn == NULL) {
      return Status(Substitute("Unable to load function '$0' from LLVM module '$1'",
          symbol, fn.hdfs_location));
    }
    // Rename the function to something more readable than the mangled name.
    string demangled_name = SymbolsUtil::DemangleNoArgs((*llvm_fn)->getName().str());
    (*llvm_fn)->setName(demangled_name);
  }
  return Status::OK();
}
int LlvmCodeGen::ReplaceCallSites(
    llvm::Function* caller, llvm::Function* new_fn, const string& target_name) {
  DCHECK(!is_compiled_);
  DCHECK(caller->getParent() == module_);
  DCHECK(caller != NULL);
  DCHECK(new_fn != NULL);
  DCHECK(find(handcrafted_functions_.begin(), handcrafted_functions_.end(), new_fn)
          == handcrafted_functions_.end()
      || finalized_functions_.find(new_fn) != finalized_functions_.end());
  vector<llvm::CallInst*> call_sites;
  FindCallSites(caller, target_name, &call_sites);
  int replaced = 0;
  for (llvm::CallInst* call_instr : call_sites) {
    // Replace the called function
    call_instr->setCalledFunction(new_fn);
    ++replaced;
  }
  return replaced;
}
int LlvmCodeGen::ReplaceCallSitesWithValue(
    llvm::Function* caller, llvm::Value* replacement, const string& target_name) {
  DCHECK(!is_compiled_);
  DCHECK(caller->getParent() == module_);
  DCHECK(caller != NULL);
  DCHECK(replacement != NULL);
  vector<llvm::CallInst*> call_sites;
  FindCallSites(caller, target_name, &call_sites);
  int replaced = 0;
  for (llvm::CallInst* call_instr : call_sites) {
    call_instr->replaceAllUsesWith(replacement);
    ++replaced;
  }
  return replaced;
}
int LlvmCodeGen::ReplaceCallSitesWithBoolConst(llvm::Function* caller, bool constant,
    const string& target_name) {
  llvm::Value* replacement = GetBoolConstant(constant);
  return ReplaceCallSitesWithValue(caller, replacement, target_name);
}
int LlvmCodeGen::InlineConstFnAttrs(const FunctionContext::TypeDesc& ret_type,
    const vector<FunctionContext::TypeDesc>& arg_types, llvm::Function* fn) {
  int replaced = 0;
  for (llvm::inst_iterator iter = inst_begin(fn), end = inst_end(fn); iter != end;) {
    // Increment iter now so we don't mess it up modifying the instruction below
    llvm::Instruction* instr = &*(iter++);
    // Look for call instructions
    if (!llvm::isa<llvm::CallInst>(instr)) continue;
    llvm::CallInst* call_instr = llvm::cast<llvm::CallInst>(instr);
    llvm::Function* called_fn = call_instr->getCalledFunction();
    // Look for call to FunctionContextImpl::GetConstFnAttr().
    if (called_fn == nullptr ||
        called_fn->getName() != FunctionContextImpl::GET_CONST_FN_ATTR_SYMBOL) {
      continue;
    }
    // 't' and 'i' arguments must be constant
    llvm::ConstantInt* t_arg =
        llvm::dyn_cast<llvm::ConstantInt>(call_instr->getArgOperand(1));
    llvm::ConstantInt* i_arg =
        llvm::dyn_cast<llvm::ConstantInt>(call_instr->getArgOperand(2));
    // This optimization is only applied to built-ins which should have constant args.
    DCHECK(t_arg != nullptr)
        << "Non-constant 't' argument to FunctionContextImpl::GetConstFnAttr()";
    DCHECK(i_arg != nullptr)
        << "Non-constant 'i' argument to FunctionContextImpl::GetConstFnAttr";
    // Replace the called function with the appropriate constant
    FunctionContextImpl::ConstFnAttr t_val =
        static_cast<FunctionContextImpl::ConstFnAttr>(t_arg->getSExtValue());
    int i_val = static_cast<int>(i_arg->getSExtValue());
    DCHECK(state_ != nullptr);
    // All supported constants are currently integers.
    call_instr->replaceAllUsesWith(GetI32Constant(FunctionContextImpl::GetConstFnAttr(
        state_->query_options().decimal_v2, state_->query_options().utf8_mode, ret_type,
        arg_types, t_val, i_val)));
    call_instr->eraseFromParent();
    ++replaced;
  }
  return replaced;
}
void LlvmCodeGen::FindCallSites(
    llvm::Function* caller, const string& target_name, vector<llvm::CallInst*>* results) {
  for (llvm::inst_iterator iter = inst_begin(caller); iter != inst_end(caller); ++iter) {
    llvm::Instruction* instr = &*iter;
    // Look for call instructions. Note that we'll ignore invoke and other related
    // instructions that are not a plain function call.
    if (llvm::CallInst::classof(instr)) {
      llvm::CallInst* call_instr = reinterpret_cast<llvm::CallInst*>(instr);
      llvm::Function* callee = call_instr->getCalledFunction();
      // Check for substring match.
      if (callee != NULL && callee->getName().find(target_name) != string::npos) {
        results->push_back(call_instr);
      }
    }
  }
}
llvm::Function* LlvmCodeGen::CloneFunction(llvm::Function* fn) {
  DCHECK(!is_compiled_);
  llvm::ValueToValueMapTy dummy_vmap;
  // Verifies that 'fn' has been materialized already. Callers are expected to use
  // GetFunction() to obtain the Function object.
  DCHECK(!fn->isMaterializable());
  // CloneFunction() automatically gives the new function a unique name
  llvm::Function* fn_clone = llvm::CloneFunction(fn, dummy_vmap);
  fn_clone->copyAttributesFrom(fn);
  return fn_clone;
}
llvm::Function* LlvmCodeGen::FinalizeFunction(llvm::Function* function) {
  SetCPUAttrs(function);
  if (!VerifyFunction(function)) return NULL;
  finalized_functions_.insert(function);
  if (FLAGS_dump_ir) {
    string fn_name = function->getName();
    LOG(INFO) << "Dump of Function "<< fn_name << ": " << LlvmCodeGen::Print(function);
  }
  return function;
}
Status LlvmCodeGen::MaterializeModule() {
  llvm::Error err = module_->materializeAll();
  if (UNLIKELY(err)) {
    string err_string;
    llvm::handleAllErrors(
        move(err), [&](llvm::ErrorInfoBase& eib) { err_string = eib.message(); });
    return Status(Substitute("Failed to materialize module $0: $1",
        module_->getName().str(), err_string));
  }
  return Status::OK();
}
// It's okay to call this function even if the module has been materialized.
Status LlvmCodeGen::FinalizeLazyMaterialization() {
  for (llvm::Function& fn : module_->functions()) {
    if (fn.isMaterializable()) {
      DCHECK(!module_->isMaterialized());
      // Unmaterialized functions can still have their declarations around. LLVM asserts
      // these unmaterialized functions' linkage types are external / external weak.
      fn.setLinkage(llvm::Function::ExternalLinkage);
      // DCE may claim the personality function is still referenced by unmaterialized
      // functions when it is deleted by DCE. Similarly, LLVM may complain if comdats
      // reference unmaterialized functions but their definition cannot be found.
      // Since the unmaterialized functions are not used anyway, just remove their
      // personality functions and comdats.
      fn.setPersonalityFn(NULL);
      fn.setComdat(NULL);
      fn.setIsMaterializable(false);
    }
  }
  // All unused functions are now not materializable so it should be quick to call
  // materializeAll(). We need to call this function in order to destroy the
  // materializer so that DCE will not assert fail.
  return MaterializeModule();
}
bool LlvmCodeGen::LookupCache(CodeGenCacheKey& cache_key) {
  DCHECK(!cache_key.empty());
  CodeGenCacheEntry entry;
  CodeGenCache* cache = ExecEnv::GetInstance()->codegen_cache();
  DCHECK(cache != nullptr);
  Status lookup_status = cache->Lookup(cache_key,
      state_->query_options().codegen_cache_mode, &entry, &engine_cache_cached_);
  bool entry_exists = lookup_status.ok() && !entry.Empty();
  LOG(INFO) << DebugCacheEntryString(cache_key, true /*is_lookup*/,
      CodeGenCacheModeAnalyzer::is_debug(state_->query_options().codegen_cache_mode),
      entry_exists);
  if (entry_exists) {
    // Fallback to normal procedure if function names hashcode is not expected.
    // The names hashcode should be the same unless there is a collision on the
    // key, we expect this case is very rare.
    if (function_names_hashcode_ != entry.function_names_hashcode) {
      LOG(WARNING)
          << "The codegen cache entry contains a different function names hashcode: "
          << " function names hashcode expected: " << function_names_hashcode_
          << " actual: " << entry.function_names_hashcode
          << " key hash_code=" << cache_key.hash_code();
      cache->IncHitOrMissCount(/*hit*/ false);
      return false;
    }
    if (entry.opt_level < state_->query_options().codegen_opt_level) {
      // Requested optimization level is higher than cached entry, so treat as a miss.
      VLOG(2) << "Overwriting codegen cache entry at " << entry.opt_level
          << " with optimization level " << state_->query_options().codegen_opt_level;
      cache->IncHitOrMissCount(/*hit*/ false);
      return false;
    }
    // Because we cache all the compiled codegened functions, the cached number of
    // functions should be the same as the total optimized function number.
    COUNTER_SET(num_cached_functions_, entry.num_opt_functions);
    COUNTER_SET(num_functions_, entry.num_functions);
    COUNTER_SET(num_instructions_, entry.num_instructions);
    COUNTER_SET(num_opt_functions_, entry.num_opt_functions);
    COUNTER_SET(num_opt_instructions_, entry.num_opt_instructions);
  }
  cache->IncHitOrMissCount(/*hit*/ entry_exists);
  return entry_exists;
}
string LlvmCodeGen::GetAllFunctionNames() {
  stringstream result;
  // The way to concat would be like "function1,function2".
  // The function names are sorted in 'fns_to_jit_compile_'.
  constexpr char separator = ',';
  for (auto& entry : fns_to_jit_compile_) {
    const llvm::StringRef& fn_name = entry.first;
    result << fn_name.data() << separator;
  }
  return result.str();
}
void LlvmCodeGen::GenerateFunctionNamesHashCode() {
  string function_names = GetAllFunctionNames();
  // Use the same hash seed as the codegen cache key.
  function_names_hashcode_ = HashUtil::MurmurHash2_64(function_names.c_str(),
      function_names.length(), CodeGenCacheKeyConstructor::CODEGEN_CACHE_HASH_SEED_CONST);
}
Status LlvmCodeGen::StoreCache(CodeGenCacheKey& cache_key) {
  DCHECK(!cache_key.empty());
  Status store_status = ExecEnv::GetInstance()->codegen_cache()->Store(
      cache_key, this, state_->query_options().codegen_cache_mode,
      state_->query_options().codegen_opt_level);
  LOG(INFO) << DebugCacheEntryString(cache_key, false /*is_lookup*/,
      CodeGenCacheModeAnalyzer::is_debug(state_->query_options().codegen_cache_mode),
      store_status.ok());
  return store_status;
}
void LlvmCodeGen::PruneModule() {
  SCOPED_TIMER(function_prune_timer_);
  // Before running any other optimization passes, run the internalize pass, giving it
  // the names of all functions registered by AddFunctionToJit(), followed by the
  // global dead code elimination pass. This causes all functions not registered to be
  // JIT'd to be marked as internal, and any internal functions that are not used are
  // deleted by DCE pass. This greatly decreases compile time by removing unused code.
  llvm::ModuleAnalysisManager module_analysis_manager;
  llvm::PassBuilder pass_builder(execution_engine()->getTargetMachine());
  pass_builder.registerModuleAnalyses(module_analysis_manager);
  llvm::ModulePassManager module_pass_manager;
  module_pass_manager.addPass(
      llvm::InternalizePass([this](const llvm::GlobalValue& gv) {
        return fns_to_jit_compile_.count(gv.getName()) > 0;
      }));
  module_pass_manager.addPass(llvm::GlobalDCEPass());
  module_pass_manager.run(*module_, module_analysis_manager);
}
Status LlvmCodeGen::FinalizeModule(string* module_id) {
  DCHECK(!is_compiled_);
  is_compiled_ = true;
  if (FLAGS_unopt_module_dir.size() != 0) {
    string path = FLAGS_unopt_module_dir + "/" + id_ + "_unopt.ll";
    fstream f(path.c_str(), fstream::out | fstream::trunc);
    if (f.fail()) {
      LOG(ERROR) << "Could not save IR to: " << path;
    } else {
      f << GetIR(true);
      f.close();
      LOG(INFO) << "Saved unoptimized IR to " << path;
    }
  }
  if (is_corrupt_) return Status("Module is corrupt.");
  SCOPED_TIMER(profile_->total_time_counter());
  SCOPED_THREAD_COUNTER_MEASUREMENT(llvm_thread_counters_);
  // Clean up handcrafted functions that have not been finalized. Clean up is done by
  // deleting the function from the module. Any reference to deleted functions in the
  // module will crash LLVM and thus Impala during finalization of the module.
  stringstream ss;
  for (llvm::Function* fn : handcrafted_functions_) {
    if (finalized_functions_.find(fn) == finalized_functions_.end()) {
      ss << fn->getName().str() << "\n";
      fn->eraseFromParent();
    }
  }
  string non_finalized_fns_str = ss.str();
  if (!non_finalized_fns_str.empty()) {
    LOG(INFO) << "For query " << PrintId(state_->query_id())
              << " the following functions were not finalized and have been removed from "
                 "the module:\n"
              << non_finalized_fns_str;
  }
  // Don't waste time optimizing module if there are no functions to JIT. This can happen
  // if the codegen object is created but no functions are successfully codegen'd.
  if (fns_to_jit_compile_.empty()) {
    DestroyModule();
    return Status::OK();
  }
  RETURN_IF_ERROR(FinalizeLazyMaterialization());
  PruneModule();
  bool codegen_cache_enabled = state_->codegen_cache_enabled() && codegen_cache_enabled_;
  CodeGenCacheKey cache_key;
  bool cache_hit = false;
  if (codegen_cache_enabled) {
    string bitcode;
    SCOPED_TIMER(codegen_cache_lookup_timer_);
    {
      SCOPED_TIMER(module_bitcode_gen_timer_);
      llvm::raw_string_ostream bitcode_stream(bitcode);
      llvm::WriteBitcodeToFile(module_, bitcode_stream);
      bitcode_stream.flush();
    }
    CodeGenCacheKeyConstructor::construct(bitcode, &cache_key);
    // Generate the function names hashcode no matter the look up result, will be used
    // in the cache store process if look up failed.
    GenerateFunctionNamesHashCode();
    DCHECK(!cache_key.empty());
    // Set the module id for the use of ObjectCache.
    module_->setModuleIdentifier(cache_key.hash_code().str());
    cache_hit = LookupCache(cache_key);
  }
  if (!cache_hit) {
    // Update counters before final optimization, but after removing unused functions.
    // This gives us a rough measure of how much work the optimization and compilation
    // must do. If found in cache, counters will be restored from the cache entry.
    InstructionCounter counter;
    counter.visit(*module_);
    COUNTER_SET(num_functions_, counter.GetCount(InstructionCounter::TOTAL_FUNCTIONS));
    COUNTER_SET(num_instructions_, counter.GetCount(InstructionCounter::TOTAL_INSTS));
    if (optimizations_enabled_ && !FLAGS_disable_optimization_passes) {
      RETURN_IF_ERROR(OptimizeModule());
      counter.ResetCount();
      counter.visit(*module_);
    }
    COUNTER_SET(
        num_opt_functions_, counter.GetCount(InstructionCounter::TOTAL_FUNCTIONS));
    COUNTER_SET(num_opt_instructions_, counter.GetCount(InstructionCounter::TOTAL_INSTS));
  }
  if (FLAGS_opt_module_dir.size() != 0) {
    string path = FLAGS_opt_module_dir + "/" + id_ + "_opt.ll";
    fstream f(path.c_str(), fstream::out | fstream::trunc);
    if (f.fail()) {
      LOG(ERROR) << "Could not save IR to: " << path;
    } else {
      f << GetIR(true);
      f.close();
      LOG(INFO) << "Saved optimized IR to " << path;
    }
  }
  if (codegen_cache_enabled) {
    if (cache_hit) {
      DCHECK(engine_cache_cached_ != nullptr);
      execution_engine()->setObjectCache(engine_cache_cached_.get());
    } else {
      execution_engine()->setObjectCache(engine_cache_.get());
    }
  }
  {
    SCOPED_TIMER(compile_timer_);
    // Finalize module, which compiles all functions.
    execution_engine()->finalizeObject();
  }
  SetFunctionPointers();
  Status store_cache_status;
  if (codegen_cache_enabled && !cache_hit) {
    SCOPED_TIMER(codegen_cache_save_timer_);
    store_cache_status = StoreCache(cache_key);
  }
  // Track the memory consumed by the runtime compiled code.
  // If codegen cache is enabled, the part stored to the cache will be taken care by
  // codegen cache to track the memory consumption.
  int64_t bytes_allocated = memory_manager_->bytes_allocated();
  if (!mem_tracker_->TryConsume(bytes_allocated)) {
    const string& msg = Substitute(
        "Failed to allocate '$0' bytes for compiled code module", bytes_allocated);
    return mem_tracker_->MemLimitExceeded(NULL, msg, bytes_allocated);
  }
  memory_manager_->set_bytes_tracked(bytes_allocated);
  // Get the module id before module destruction.
  if (module_id != nullptr) *module_id = module_->getModuleIdentifier();
  DestroyModule();
  return Status::OK();
}
Status LlvmCodeGen::FinalizeModuleAsync(RuntimeProfile::EventSequence* event_sequence) {
  DCHECK(event_sequence != nullptr);
  Status thread_start_status = Thread::Create("async-codegen", "async-codegen",
      [this, event_sequence]() {
        SCOPED_THREAD_COUNTER_MEASUREMENT(compile_thread_counters_);
        VLOG(2) << "Starting async code generation.";
        Status status = DebugAction(state_->query_options(),
            "BEFORE_CODEGEN_IN_ASYNC_CODEGEN_THREAD");
        if (status.ok()) {
          status = this->FinalizeModule();
        }
        const std::string status_msg = status.ok() ? "OK." : status.msg().msg();
        auto log_level = status.ok() ? 2 : 1;
        event_sequence->MarkEvent("AsyncCodegenFinished");
        VLOG(log_level) << "Finished async code generation with result: " << status;
      }, &async_compile_thread_);
  event_sequence->MarkEvent("AsyncCodegenStarted");
  RETURN_IF_ERROR(DebugAction(state_->query_options(),
        "AFTER_STARTING_ASYNC_CODEGEN_IN_FRAGMENT_THREAD"));
  return thread_start_status;
}
/// TODO: In asynchronous mode, return early if the query is cancelled or finished.
Status LlvmCodeGen::OptimizeModule() {
  SCOPED_TIMER(optimization_timer_);
  llvm::LoopAnalysisManager LAM;
  llvm::FunctionAnalysisManager FAM;
  llvm::CGSCCAnalysisManager CGAM;
  llvm::ModuleAnalysisManager MAM;
  // This pass manager will construct optimizations passes that are "typical" for
  // c/c++ programs. We're relying on llvm to pick the best passes for us.
  // TODO: we can likely muck with this to get better compile speeds or write
  // our own passes. Our subexpression elimination optimization can be rolled into
  // a pass.
  llvm::PassBuilder pass_builder(execution_engine()->getTargetMachine());
  pass_builder.registerModuleAnalyses(MAM);
  pass_builder.registerCGSCCAnalyses(CGAM);
  pass_builder.registerFunctionAnalyses(FAM);
  pass_builder.registerLoopAnalyses(LAM);
  pass_builder.crossRegisterProxies(LAM, FAM, CGAM, MAM);
  TCodeGenOptLevel::type opt_level = state_->query_options().codegen_opt_level;
  llvm::PassBuilder::OptimizationLevel opt;
  // GCC's -Werror=switch errors if a case is not covered.
  switch (opt_level) {
    case TCodeGenOptLevel::O0:
      // Default optimization pipeline requires O1 or greater, so for O0 we skip.
      return Status::OK();
    case TCodeGenOptLevel::O1:
      opt = llvm::PassBuilder::OptimizationLevel::O1;
      break;
    case TCodeGenOptLevel::Os:
      opt = llvm::PassBuilder::OptimizationLevel::Os;
      break;
    case TCodeGenOptLevel::O2:
      opt = llvm::PassBuilder::OptimizationLevel::O2;
      break;
    case TCodeGenOptLevel::O3:
      opt = llvm::PassBuilder::OptimizationLevel::O3;
      break;
  }
  llvm::ModulePassManager pass_manager = pass_builder.buildPerModuleDefaultPipeline(opt);
  int64_t estimated_memory = ESTIMATED_OPTIMIZER_BYTES_PER_INST
      * num_instructions_->value();
  if (!mem_tracker_->TryConsume(estimated_memory)) {
    const string& msg = Substitute(
        "Codegen failed to reserve '$0' bytes for optimization", estimated_memory);
    return mem_tracker_->MemLimitExceeded(NULL, msg, estimated_memory);
  }
  // Create and run module pass manager
  pass_manager.run(*module_, MAM);
  if (FLAGS_print_llvm_ir_instruction_count) {
    for (auto& entry : fns_to_jit_compile_) {
      InstructionCounter counter;
      llvm::Function* llvm_function = entry.second.first;
      const llvm::StringRef& llvm_function_name = entry.first;
      counter.visit(*llvm_function);
      VLOG(1) << llvm_function_name.data();
      VLOG(1) << counter.PrintCounters();
    }
  }
  mem_tracker_->Release(estimated_memory);
  return Status::OK();
}
bool LlvmCodeGen::SetFunctionPointers(CodeGenCache* cache,
    const CodeGenCacheKey* cache_key) {
  // Get pointers to all codegen'd functions.
  for (auto& entry : fns_to_jit_compile_) {
    const llvm::StringRef& function_name = entry.first;
    LlvmFunctionWithFnPtrTargets& fn_with_targets = entry.second;
    llvm::Function* function = fn_with_targets.first;
    std::vector<CodegenFnPtrBase*>& jitted_fn_ptrs = fn_with_targets.second;
    void* jitted_function = nullptr;
    if (cache != nullptr) {
      DCHECK(cache_key != nullptr);
      // engine_cache_cached_ is used to keep the life of the object cache
      // in case the object cache is evicted in the global cache.
      DCHECK(engine_cache_cached_ != nullptr);
      // Using the function getFunctionAddress() with a non-existent function name would
      // hit an assertion during the test, could be a bug in llvm 5, need to review after
      // upgrade llvm. But because we already checked the names hashcode for key collision
      // cases, we expect all the functions should be in the cached execution engine.
      jitted_function =
          reinterpret_cast<void*>(execution_engine()->getFunctionAddress(function_name));
      if (jitted_function == nullptr) {
        LOG(WARNING) << "Failed to get a jitted function from cache: "
                     << function_name.data()
                     << " key hash_code=" << cache_key->hash_code();
        cache->IncHitOrMissCount(/*hit*/ false);
        return false;
      }
    } else {
      DCHECK(cache_key == nullptr);
      jitted_function = execution_engine()->getPointerToFunction(function);
      DCHECK(jitted_function != nullptr) << "Failed to jit " << function_name.data();
    }
    DCHECK(jitted_function != nullptr);
    for (CodegenFnPtrBase* jitted_fn_ptr : jitted_fn_ptrs) {
      jitted_fn_ptr->store(jitted_function);
    }
  }
  return true;
}
void LlvmCodeGen::DestroyModule() {
  // Clear all references to LLVM objects owned by the module.
  cross_compiled_functions_.clear();
  handcrafted_functions_.clear();
  registered_exprs_map_.clear();
  registered_exprs_.clear();
  llvm_intrinsics_.clear();
  hash_fns_.clear();
  fns_to_jit_compile_.clear();
  execution_engine()->removeModule(module_);
  module_ = NULL;
}
void LlvmCodeGen::AddFunctionToJit(llvm::Function* fn, CodegenFnPtrBase* fn_ptr) {
  DCHECK(finalized_functions_.find(fn) != finalized_functions_.end())
      << "Attempted to add a non-finalized function to Jit: " << fn->getName().str();
  DCHECK(!is_compiled_);
  llvm::Type* decimal_val_type = GetNamedType(CodegenAnyVal::LLVM_DECIMALVAL_NAME);
  if (fn->getReturnType() == decimal_val_type) {
    // Per the x86 calling convention ABI, DecimalVals should be returned via an extra
    // first DecimalVal* argument. We generate non-compliant functions that return the
    // DecimalVal directly, which we can call from generated code, but not from compiled
    // native code. To avoid accidentally calling a non-compliant function from native
    // code, call 'function' from an ABI-compliant wrapper.
    stringstream name;
    name << fn->getName().str() << "ABIWrapper";
    LlvmCodeGen::FnPrototype prototype(this, name.str(), void_type_);
    // Add return argument
    prototype.AddArgument(NamedVariable("result", decimal_val_type->getPointerTo()));
    // Add regular arguments
    for (llvm::Function::arg_iterator arg = fn->arg_begin(); arg != fn->arg_end();
         ++arg) {
      prototype.AddArgument(NamedVariable(arg->getName(), arg->getType()));
    }
    LlvmBuilder builder(context());
    llvm::Value* args[fn->arg_size() + 1];
    llvm::Function* fn_wrapper = prototype.GeneratePrototype(&builder, &args[0]);
    fn_wrapper->addFnAttr(llvm::Attribute::AlwaysInline);
    // Mark first argument as sret (not sure if this is necessary but it can't hurt)
    fn_wrapper->addAttribute(1, llvm::Attribute::StructRet);
    // Call 'fn' and store the result in the result argument
    llvm::Value* result = builder.CreateCall(
        fn, llvm::ArrayRef<llvm::Value*>({&args[1], fn->arg_size()}), "result");
    builder.CreateStore(result, args[0]);
    builder.CreateRetVoid();
    fn = FinalizeFunction(fn_wrapper);
    DCHECK(fn != NULL);
  }
  AddFunctionToJitInternal(fn, fn_ptr);
}
void LlvmCodeGen::AddFunctionToJitInternal(llvm::Function* fn, CodegenFnPtrBase* fn_ptr) {
  DCHECK(fn != nullptr);
  DCHECK(fn_ptr != nullptr);
  const llvm::StringRef& fn_name = fn->getName();
  auto it = fns_to_jit_compile_.find(fn_name);
  if (it == fns_to_jit_compile_.end()) {
    fns_to_jit_compile_[fn_name] = make_pair(fn, vector<CodegenFnPtrBase*>{fn_ptr});
  } else {
    it->second.second.push_back(fn_ptr);
  }
}
void LlvmCodeGen::CodegenDebugTrace(
    LlvmBuilder* builder, const char* str, llvm::Value* v1) {
  LOG(ERROR) << "Remove IR codegen debug traces before checking in.";
  // Call printf by embedding the string into the module and getting a pointer to it.
  llvm::Value* const llvm_str =
      GetStringConstant(builder, Substitute("LLVM Trace: $0", str));
  llvm::Function* printf = module_->getFunction("printf");
  DCHECK(printf != nullptr);
  vector<llvm::Value*> calling_args;
  calling_args.push_back(llvm_str);
  if (v1 != NULL) calling_args.push_back(v1);
  builder->CreateCall(printf, calling_args);
}
Status LlvmCodeGen::GetSymbols(const string& file, const string& module_id,
    unordered_set<string>* symbols) {
  ObjectPool pool;
  scoped_ptr<LlvmCodeGen> codegen;
  RETURN_IF_ERROR(CreateFromFile(nullptr, &pool, nullptr, file, module_id, &codegen));
  for (const llvm::Function& fn : codegen->module_->functions()) {
    if (fn.isMaterializable()) symbols->insert(fn.getName());
  }
  codegen->Close();
  return Status::OK();
}
// TODO: cache this function (e.g. all min(int, int) are identical).
// we probably want some more global IR function cache, or, implement this
// in c and precompile it with clang.
// define i32 @Min(i32 %v1, i32 %v2) {
// entry:
// %0 = icmp slt i32 %v1, %v2
// br i1 %0, label %ret_v1, label %ret_v2
//
// ret_v1: ; preds = %entry
// ret i32 %v1
//
// ret_v2: ; preds = %entry
// ret i32 %v2
// }
void LlvmCodeGen::CodegenMinMax(LlvmBuilder* builder, const ColumnType& type,
    llvm::Value* src, llvm::Value* dst_slot_ptr, bool min, llvm::Function* fn) {
  llvm::Value* dst = builder->CreateLoad(dst_slot_ptr, "dst_val");
  llvm::Value* compare = NULL;
  switch (type.type) {
    case TYPE_NULL:
      compare = false_value();
      break;
    case TYPE_BOOLEAN:
      if (min) {
        // For min, return x && y
        compare = builder->CreateAnd(src, dst);
      } else {
        // For max, return x || y
        compare = builder->CreateOr(src, dst);
      }
      break;
    case TYPE_TINYINT:
    case TYPE_SMALLINT:
    case TYPE_INT:
    case TYPE_DATE:
    case TYPE_BIGINT:
    case TYPE_DECIMAL:
      if (min) {
        compare = builder->CreateICmpSLT(src, dst);
      } else {
        compare = builder->CreateICmpSGT(src, dst);
      }
      break;
    case TYPE_FLOAT:
    case TYPE_DOUBLE:
      if (min) {
        // OLT is true if 'src' < 'dst' and neither 'src' nor 'dst' is 'nan'.
        compare = builder->CreateFCmpOLT(src, dst);
      } else {
        // OGT is true if 'src' > 'dst' and neither 'src' nor 'dst' is 'nan'.
        compare = builder->CreateFCmpOGT(src, dst);
      }
      // UNE is true if the operands are not equal or if either operand is a 'nan'. Since
      // we're comparing 'src' to itself, the UNE will only be true if 'src' is 'nan'.
      compare = builder->CreateOr(compare, builder->CreateFCmpUNE(src, src));
      break;
    default:
      DCHECK(false);
  }
  if (type.type == TYPE_BOOLEAN) {
    builder->CreateStore(compare, dst_slot_ptr);
  } else {
    llvm::BasicBlock *ret_v1, *ret_v2;
    CreateIfElseBlocks(fn, "ret_v1", "ret_v2", &ret_v1, &ret_v2);
    builder->CreateCondBr(compare, ret_v1, ret_v2);
    builder->SetInsertPoint(ret_v1);
    builder->CreateStore(src, dst_slot_ptr);
    builder->CreateBr(ret_v2);
    builder->SetInsertPoint(ret_v2);
  }
}
// Intrinsics are loaded one by one. Some are overloaded (e.g. memcpy) and the types must
// be specified.
// TODO: is there a better way to do this?
Status LlvmCodeGen::LoadIntrinsics() {
  // Load memcpy
  {
    llvm::Type* types[] = {ptr_type(), ptr_type(), i32_type()};
    llvm::Function* fn =
        llvm::Intrinsic::getDeclaration(module_, llvm::Intrinsic::memcpy, types);
    if (fn == NULL) {
      return Status("Could not find memcpy intrinsic.");
    }
    llvm_intrinsics_[llvm::Intrinsic::memcpy] = fn;
  }
  // TODO: where is the best place to put this?
  struct {
    llvm::Intrinsic::ID id;
    const char* error;
  } non_overloaded_intrinsics[] = {
#ifdef __aarch64__
      {llvm::Intrinsic::aarch64_crc32cb, "aarch64 crc32_u8"},
      {llvm::Intrinsic::aarch64_crc32ch, "aarch64 crc32_u16"},
      {llvm::Intrinsic::aarch64_crc32cw, "aarch64 crc32_u32"},
      {llvm::Intrinsic::aarch64_crc32cx, "aarch64 crc32_u64"},
#else
      {llvm::Intrinsic::x86_sse42_crc32_32_8, "sse4.2 crc32_u8"},
      {llvm::Intrinsic::x86_sse42_crc32_32_16, "sse4.2 crc32_u16"},
      {llvm::Intrinsic::x86_sse42_crc32_32_32, "sse4.2 crc32_u32"},
      {llvm::Intrinsic::x86_sse42_crc32_64_64, "sse4.2 crc32_u64"},
#endif
  };
  const int num_intrinsics =
      sizeof(non_overloaded_intrinsics) / sizeof(non_overloaded_intrinsics[0]);
  for (int i = 0; i < num_intrinsics; ++i) {
    llvm::Intrinsic::ID id = non_overloaded_intrinsics[i].id;
    llvm::Function* fn = llvm::Intrinsic::getDeclaration(module_, id);
    if (fn == NULL) {
      stringstream ss;
      ss << "Could not find " << non_overloaded_intrinsics[i].error << " intrinsic";
      return Status(ss.str());
    }
    llvm_intrinsics_[id] = fn;
  }
  return Status::OK();
}
void LlvmCodeGen::CodegenMemcpy(
    LlvmBuilder* builder, llvm::Value* dst, llvm::Value* src, int size) {
  DCHECK_GE(size, 0);
  if (size == 0) return;
  llvm::Value* size_val = GetI64Constant(size);
  CodegenMemcpy(builder, dst, src, size_val);
}
void LlvmCodeGen::CodegenMemcpy(
    LlvmBuilder* builder, llvm::Value* dst, llvm::Value* src, llvm::Value* size) {
  DCHECK(dst->getType()->isPointerTy()) << Print(dst);
  DCHECK(src->getType()->isPointerTy()) << Print(src);
  builder->CreateMemCpy(dst, src, size, /* no alignment */ 0);
}
void LlvmCodeGen::CodegenMemset(
    LlvmBuilder* builder, llvm::Value* dst, int value, int size) {
  DCHECK(dst->getType()->isPointerTy()) << Print(dst);
  DCHECK_GE(size, 0);
  if (size == 0) return;
  llvm::Value* value_const = GetI8Constant(value);
  builder->CreateMemSet(dst, value_const, size, /* no alignment */ 0);
}
void LlvmCodeGen::CodegenClearNullBits(
    LlvmBuilder* builder, llvm::Value* tuple_ptr, const TupleDescriptor& tuple_desc) {
  llvm::Value* int8_ptr = builder->CreateBitCast(tuple_ptr, ptr_type(), "int8_ptr");
  llvm::Value* null_bytes_offset = GetI32Constant(tuple_desc.null_bytes_offset());
  llvm::Value* null_bytes_ptr =
      builder->CreateInBoundsGEP(int8_ptr, null_bytes_offset, "null_bytes_ptr");
  CodegenMemset(builder, null_bytes_ptr, 0, tuple_desc.num_null_bytes());
}
llvm::Value* LlvmCodeGen::CodegenMemPoolAllocate(LlvmBuilder* builder,
    llvm::Value* pool_val, llvm::Value* size_val, const char* name) {
  DCHECK(pool_val != nullptr);
  DCHECK(size_val->getType()->isIntegerTy());
  DCHECK_LE(size_val->getType()->getIntegerBitWidth(), 64);
  DCHECK_EQ(pool_val->getType(), GetStructPtrType<MemPool>());
  // Extend 'size_val' to i64 if necessary
  if (size_val->getType()->getIntegerBitWidth() < 64) {
    size_val = builder->CreateSExt(size_val, i64_type());
  }
  llvm::Function* allocate_fn = GetFunction(IRFunction::MEMPOOL_ALLOCATE, false);
  llvm::Value* alignment = GetI32Constant(MemPool::DEFAULT_ALIGNMENT);
  llvm::Value* fn_args[] = {pool_val, size_val, alignment};
  return builder->CreateCall(allocate_fn, fn_args, name);
}
llvm::Value* LlvmCodeGen::CodegenArrayAt(
    LlvmBuilder* builder, llvm::Value* array, int idx, const char* name) {
  DCHECK(array->getType()->isPointerTy() || array->getType()->isArrayTy())
      << Print(array->getType());
  llvm::Value* ptr = builder->CreateConstGEP1_32(array, idx);
  return builder->CreateLoad(ptr, name);
}
llvm::Value* LlvmCodeGen::CodegenCallFunction(LlvmBuilder* builder,
    IRFunction::Type ir_type, llvm::ArrayRef<llvm::Value*> args, const char* name) {
  llvm::Function* fn = GetFunction(ir_type, false);
  DCHECK(fn != nullptr);
  return builder->CreateCall(fn, args, name);
}
void LlvmCodeGen::ClearHashFns() {
  hash_fns_.clear();
}
// Codegen to compute hash for a particular byte size. Loops are unrolled in this
// process. For the case where num_bytes == 11, we'd do this by calling
// 1. crc64 (for first 8 bytes)
// 2. crc16 (for bytes 9, 10)
// 3. crc8 (for byte 11)
// The resulting IR looks like:
// define i32 @CrcHash11(i8* %data, i32 %len, i32 %seed) {
// entry:
// %0 = zext i32 %seed to i64
// %1 = bitcast i8* %data to i64*
// %2 = getelementptr i64* %1, i32 0
// %3 = load i64* %2
// %4 = call i64 @llvm.x86.sse42.crc32.64.64(i64 %0, i64 %3)
// %5 = trunc i64 %4 to i32
// %6 = getelementptr i8* %data, i32 8
// %7 = bitcast i8* %6 to i16*
// %8 = load i16* %7
// %9 = call i32 @llvm.x86.sse42.crc32.32.16(i32 %5, i16 %8)
// %10 = getelementptr i8* %6, i32 2
// %11 = load i8* %10
// %12 = call i32 @llvm.x86.sse42.crc32.32.8(i32 %9, i8 %11)
// ret i32 %12
// }
llvm::Function* LlvmCodeGen::GetHashFunction(int num_bytes) {
  if (IS_AARCH64 || IsCPUFeatureEnabled(CpuInfo::SSE4_2)) {
    if (num_bytes == -1) {
      // -1 indicates variable length, just return the generic loop based
      // hash fn.
      return GetFunction(IRFunction::HASH_CRC, false);
    }
    map<int, llvm::Function*>::iterator cached_fn = hash_fns_.find(num_bytes);
    if (cached_fn != hash_fns_.end()) {
      return cached_fn->second;
    }
    // Generate a function to hash these bytes
    stringstream ss;
    ss << "CrcHash" << num_bytes;
    FnPrototype prototype(this, ss.str(), i32_type());
    prototype.AddArgument(LlvmCodeGen::NamedVariable("data", ptr_type()));
    prototype.AddArgument(LlvmCodeGen::NamedVariable("len", i32_type()));
    prototype.AddArgument(LlvmCodeGen::NamedVariable("seed", i32_type()));
    llvm::Value* args[3];
    LlvmBuilder builder(context());
    llvm::Function* fn = prototype.GeneratePrototype(&builder, &args[0]);
    llvm::Value* data = args[0];
    llvm::Value* result = args[2];
#ifdef __aarch64__
    llvm::Function* crc8_fn = llvm_intrinsics_[llvm::Intrinsic::aarch64_crc32cb];
    llvm::Function* crc16_fn = llvm_intrinsics_[llvm::Intrinsic::aarch64_crc32ch];
    llvm::Function* crc32_fn = llvm_intrinsics_[llvm::Intrinsic::aarch64_crc32cw];
    llvm::Function* crc64_fn = llvm_intrinsics_[llvm::Intrinsic::aarch64_crc32cx];
#else
    llvm::Function* crc8_fn = llvm_intrinsics_[llvm::Intrinsic::x86_sse42_crc32_32_8];
    llvm::Function* crc16_fn = llvm_intrinsics_[llvm::Intrinsic::x86_sse42_crc32_32_16];
    llvm::Function* crc32_fn = llvm_intrinsics_[llvm::Intrinsic::x86_sse42_crc32_32_32];
    llvm::Function* crc64_fn = llvm_intrinsics_[llvm::Intrinsic::x86_sse42_crc32_64_64];
#endif
    // Generate the crc instructions starting with the highest number of bytes
    if (num_bytes >= 8) {
#ifndef __aarch64__
      llvm::Value* result_64 = builder.CreateZExt(result, i64_type());
#endif
      llvm::Value* ptr = builder.CreateBitCast(data, i64_ptr_type());
      int i = 0;
      while (num_bytes >= 8) {
        llvm::Value* index[] = {GetI32Constant(i++)};
        llvm::Value* d = builder.CreateLoad(builder.CreateInBoundsGEP(ptr, index));
#ifdef __aarch64__
        result = builder.CreateCall(crc64_fn, llvm::ArrayRef<llvm::Value*>({result, d}));
#else
        result_64 =
            builder.CreateCall(crc64_fn, llvm::ArrayRef<llvm::Value*>({result_64, d}));
#endif
        num_bytes -= 8;
      }
#ifndef __aarch64__
      result = builder.CreateTrunc(result_64, i32_type());
#endif
      llvm::Value* index[] = {GetI32Constant(i * 8)};
      // Update data to past the 8-byte chunks
      data = builder.CreateInBoundsGEP(data, index);
    }
    if (num_bytes >= 4) {
      DCHECK_LT(num_bytes, 8);
      llvm::Value* ptr = builder.CreateBitCast(data, i32_ptr_type());
      llvm::Value* d = builder.CreateLoad(ptr);
      result = builder.CreateCall(crc32_fn, llvm::ArrayRef<llvm::Value*>({result, d}));
      llvm::Value* index[] = {GetI32Constant(4)};
      data = builder.CreateInBoundsGEP(data, index);
      num_bytes -= 4;
    }
    if (num_bytes >= 2) {
      DCHECK_LT(num_bytes, 4);
      llvm::Value* ptr = builder.CreateBitCast(data, i16_ptr_type());
      llvm::Value* d = builder.CreateLoad(ptr);
#ifdef __aarch64__
      d = builder.CreateZExt(d, i32_type());
#endif
      result = builder.CreateCall(crc16_fn, llvm::ArrayRef<llvm::Value*>({result, d}));
      llvm::Value* index[] = {GetI16Constant(2)};
      data = builder.CreateInBoundsGEP(data, index);
      num_bytes -= 2;
    }
    if (num_bytes > 0) {
      DCHECK_EQ(num_bytes, 1);
      llvm::Value* d = builder.CreateLoad(data);
#ifdef __aarch64__
      d = builder.CreateZExt(d, i32_type());
#endif
      result = builder.CreateCall(crc8_fn, llvm::ArrayRef<llvm::Value*>({result, d}));
      --num_bytes;
    }
    DCHECK_EQ(num_bytes, 0);
    llvm::Value* shift_16 = GetI32Constant(16);
    llvm::Value* upper_bits = builder.CreateShl(result, shift_16);
    llvm::Value* lower_bits = builder.CreateLShr(result, shift_16);
    result = builder.CreateOr(upper_bits, lower_bits);
    builder.CreateRet(result);
    fn = FinalizeFunction(fn);
    if (fn != NULL) {
      hash_fns_[num_bytes] = fn;
    }
    return fn;
  } else {
    return GetMurmurHashFunction(num_bytes);
  }
}
static llvm::Function* GetLenOptimizedHashFn(
    LlvmCodeGen* codegen, IRFunction::Type f, int len) {
  llvm::Function* fn = codegen->GetFunction(f, false);
  DCHECK(fn != NULL);
  if (len != -1) {
    // Clone this function since we're going to modify it by replacing the
    // length with num_bytes.
    fn = codegen->CloneFunction(fn);
    llvm::Value* len_arg = codegen->GetArgument(fn, 1);
    len_arg->replaceAllUsesWith(codegen->GetI32Constant(len));
  }
  return codegen->FinalizeFunction(fn);
}
llvm::Function* LlvmCodeGen::GetMurmurHashFunction(int len) {
  return GetLenOptimizedHashFn(this, IRFunction::HASH_MURMUR, len);
}
void LlvmCodeGen::ReplaceInstWithValue(llvm::Instruction* from, llvm::Value* to) {
  llvm::BasicBlock::iterator iter(from);
  llvm::ReplaceInstWithValue(from->getParent()->getInstList(), iter, to);
}
llvm::Argument* LlvmCodeGen::GetArgument(llvm::Function* fn, int i) {
  DCHECK_LE(i, fn->arg_size());
  llvm::Function::arg_iterator iter = fn->arg_begin();
  for (int j = 0; j < i; ++j) ++iter;
  return &*iter;
}
llvm::Value* LlvmCodeGen::GetPtrTo(
    LlvmBuilder* builder, llvm::Value* v, const char* name) {
  llvm::Value* ptr = CreateEntryBlockAlloca(*builder, v->getType(), name);
  builder->CreateStore(v, ptr);
  return ptr;
}
llvm::Constant* LlvmCodeGen::ConstantToGVPtr(
    llvm::Type* type, llvm::Constant* ir_constant, const string& name) {
  llvm::GlobalVariable* gv = new llvm::GlobalVariable(
      *module_, type, true, llvm::GlobalValue::PrivateLinkage, ir_constant, name);
  return llvm::ConstantExpr::getGetElementPtr(
      NULL, gv, llvm::ArrayRef<llvm::Constant*>({GetI32Constant(0)}));
}
llvm::Constant* LlvmCodeGen::ConstantsToGVArrayPtr(llvm::Type* element_type,
    llvm::ArrayRef<llvm::Constant*> ir_constants, const string& name) {
  llvm::ArrayType* array_type = llvm::ArrayType::get(element_type, ir_constants.size());
  llvm::Constant* array_const = llvm::ConstantArray::get(array_type, ir_constants);
  return ConstantToGVPtr(array_type, array_const, name);
}
std::unordered_set<string> LlvmCodeGen::ApplyCpuAttrWhitelist(
    const std::unordered_set<string>& cpu_attrs) {
  std::unordered_set<string> result;
  vector<string> attr_whitelist;
  boost::split(attr_whitelist, FLAGS_llvm_cpu_attr_whitelist, boost::is_any_of(","));
  for (const string& attr : cpu_attrs) {
    DCHECK_GE(attr.size(), 1);
    DCHECK(attr[0] == '-' || attr[0] == '+') << attr;
    if (attr[0] == '-') {
      // Already disabled - copy it over unmodified.
      result.insert(attr);
      continue;
    }
    const string attr_name = attr.substr(1);
    auto it = std::find(attr_whitelist.begin(), attr_whitelist.end(), attr_name);
    if (it != attr_whitelist.end()) {
      // In whitelist - copy it over unmodified.
      result.insert(attr);
    } else {
      // Not in whitelist - disable it.
      result.insert("-" + attr_name);
    }
  }
  return result;
}
void LlvmCodeGen::DiagnosticHandler::DiagnosticHandlerFn(
    const llvm::DiagnosticInfo& info, void* context) {
  if (info.getSeverity() == llvm::DiagnosticSeverity::DS_Error) {
    LlvmCodeGen* codegen = reinterpret_cast<LlvmCodeGen*>(context);
    codegen->diagnostic_handler_.error_str_.clear();
    llvm::raw_string_ostream error_msg(codegen->diagnostic_handler_.error_str_);
    llvm::DiagnosticPrinterRawOStream diagnostic_printer(error_msg);
    diagnostic_printer << "LLVM diagnostic error: ";
    info.print(diagnostic_printer);
    error_msg.flush();
    if (codegen->state_) {
      LOG(INFO) << "Query " << PrintId(codegen->state_->query_id()) << " encountered a "
          << codegen->diagnostic_handler_.error_str_;
    }
  }
}
string LlvmCodeGen::DiagnosticHandler::GetErrorString() {
  if (!error_str_.empty()) {
    string return_msg(move(error_str_)); // Also clears error_str_.
    return return_msg;
  }
  return "";
}
string LlvmCodeGen::DebugCacheEntryString(CodeGenCacheKey& key, bool is_lookup,
    bool debug_mode, bool success) const {
  stringstream out;
  if (is_lookup) {
    out << "Look up codegen cache ";
  } else {
    out << "Store to codegen cache ";
  }
  if (success) {
    out << "succeeded. ";
  } else {
    if (is_lookup) {
      out << "missed. ";
    } else {
      out << "failed. ";
    }
  }
  out << "CodeGen Cache Key hash_code=" << key.hash_code();
  if (UNLIKELY(debug_mode)) {
    out << "\nFragment Plan: " << apache::thrift::ThriftDebugString(state_->fragment())
        << "\n";
    out << "CodeGen Functions: \n";
    for (auto& entry : fns_to_jit_compile_) {
      const llvm::StringRef& fn_name = entry.first;
      out << " " << fn_name.data() << "\n";
    }
  }
  return out.str();
}
}
namespace boost {
/// Handler for exceptions in cross-compiled functions.
/// When boost is configured with BOOST_NO_EXCEPTIONS, it calls these handlers instead of
/// throwing the exception.
[[noreturn]] void throw_exception(std::exception const& e) {
  LOG(FATAL) << "Cannot handle exceptions in codegen'd code " << e.what();
}
[[noreturn]] void throw_exception(
    std::exception const& e, boost::source_location const& loc) {
  LOG(FATAL) << loc.file_name() << ":" << loc.line() << "] " << loc.function_name()
             << ": Cannot handle exceptions in codegen'd code " << e.what();
}
}

```cpp
 // Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.


#ifndef IMPALA_EXPRS_AGG_FN_H
#define IMPALA_EXPRS_AGG_FN_H

#include "exprs/expr.h"
#include "runtime/descriptors.h"
#include "udf/udf.h"

namespace llvm {
class Function;
}

namespace impala {

using impala_udf::FunctionContext;

class FragmentState;
class LlvmCodeGen;
class MemPool;
class MemTracker;
class ObjectPool;
class RuntimeState;
class Tuple;
class TupleRow;
class TExprNode;

/// --- AggFn overview
///
/// An aggregate function generates an output over a set of tuple rows.
/// An example would be AVG() which computes the average of all input rows.
/// The built-in aggregate functions such as min, max, sum, avg, ndv etc are
/// in this category.
///
/// --- Implementation
///
/// AggFn contains the aggregation operations, pointers to the UDAF interface functions
/// implementing various states of aggregation and the descriptors for the intermediate
/// and output values. Please see udf/udf.h for details of the UDAF interfaces.
///
/// AggFnEvaluator is the interface for evaluating aggregate functions against input
/// tuple rows. It invokes the following functions at different phases of the aggregation:
///
/// init_fn_     : An initialization function that initializes the aggregate value.
///
/// update_fn_   : An update function that processes the arguments for each row in the
///                query result set and accumulates an intermediate result. For example,
///                this function might increment a counter, append to a string buffer or
///                add the input to a culmulative sum.
///
/// merge_fn_    : A merge function that combines multiple intermediate results into a
///                single value.
///
/// serialize_fn_: A serialization function that flattens any intermediate values
///                containing pointers, and frees any memory allocated during the init,
///                update and merge phases.
///
/// finalize_fn_ : A finalize function that either passes through the combined result
///                unchanged, or does one final transformation. Also frees the resources
///                allocated during init, update and merge phases.
///
/// get_value_fn_: Used by AnalyticEval node to obtain the current intermediate value.
///
/// remove_fn_   : Used by AnalyticEval node to undo the update to the intermediate value
///                by an input row as it falls out of a sliding window.
///
class AggFn : public Expr {
 public:

  /// Override the base class' implementation.
  virtual bool IsAggFn() const { return true; }

  /// Enum for some built-in aggregation ops.
  enum AggregationOp {
    COUNT,
    MIN,
    MAX,
    SUM,
    AVG,
    NDV,
    OTHER,
  };

  /// Creates and initializes an aggregate function from 'texpr' and returns it in
  /// 'agg_fn'. The returned AggFn lives in the ObjectPool of 'state'. 'row_desc' is
  /// the row descriptor of the input tuple row; 'intermediate_slot_desc' is the slot
  /// descriptor of the intermediate value; 'output_slot_desc' is the slot descriptor
  /// of the output value. On failure, returns error status and sets 'agg_fn' to NULL.
  static Status Create(const TExpr& texpr, const RowDescriptor& row_desc,
      const SlotDescriptor& intermediate_slot_desc,
      const SlotDescriptor& output_slot_desc, FragmentState* state,
      AggFn** agg_fn) WARN_UNUSED_RESULT;

  bool is_merge() const { return is_merge_; }
  AggregationOp agg_op() const { return agg_op_; }
  bool is_count_star() const { return agg_op_ == COUNT && children_.empty(); }
  bool is_builtin() const { return fn_.binary_type == TFunctionBinaryType::BUILTIN; }
  const std::string& fn_name() const { return fn_.name.function_name; }
  const ColumnType& intermediate_type() const { return intermediate_slot_desc_.type(); }
  const SlotDescriptor& intermediate_slot_desc() const { return intermediate_slot_desc_; }
  // Output type is the same as Expr::type().
  const SlotDescriptor& output_slot_desc() const { return output_slot_desc_; }
  void* remove_fn() const { return remove_fn_; }
  void* merge_or_update_fn() const { return is_merge_ ? merge_fn_ : update_fn_; }
  void* serialize_fn() const { return serialize_fn_; }
  void* get_value_fn() const { return get_value_fn_; }
  void* finalize_fn() const { return finalize_fn_; }
  bool SupportsRemove() const { return remove_fn_ != nullptr; }
  bool SupportsSerialize() const { return serialize_fn_ != nullptr; }
  FunctionContext::TypeDesc GetIntermediateTypeDesc() const;
  FunctionContext::TypeDesc GetOutputTypeDesc() const;
  const std::vector<FunctionContext::TypeDesc>& arg_type_descs() const {
    return arg_type_descs_;
  }

  /// Generates an IR wrapper function to call update_fn_/merge_fn_ which may either be
  /// cross-compiled or loaded from an external library. The generated IR function is
  /// returned in 'uda_fn'. Returns error status on failure.
  /// TODO: implement codegen path for init, finalize, serialize functions etc.
  Status CodegenUpdateOrMergeFunction(LlvmCodeGen* codegen, llvm::Function** uda_fn)
      WARN_UNUSED_RESULT;

  /// Releases all cache entries to libCache for all nodes in the expr tree.
  virtual void Close();
  static void Close(const std::vector<AggFn*>& exprs);

  virtual std::string DebugString() const;
  static std::string DebugString(const std::vector<AggFn*>& exprs);

 private:
  friend class Expr;
  friend class AggFnEvaluator;

  /// True if this is a merging aggregation.
  const bool is_merge_;

  /// Slot into which Update()/Merge()/Serialize() write their result. Not owned.
  const SlotDescriptor& intermediate_slot_desc_;

  /// Slot into which Finalize() results are written. Not owned. Identical to
  /// intermediate_slot_desc_ if this agg fn has the same intermediate and result type.
  const SlotDescriptor& output_slot_desc_;

  /// The types of the arguments to the aggregate function.
  const std::vector<FunctionContext::TypeDesc> arg_type_descs_;

  /// The aggregation operation.
  AggregationOp agg_op_;

  /// Function pointers for the different phases of the aggregate function.
  void* init_fn_ = nullptr;
  void* update_fn_ = nullptr;
  void* remove_fn_ = nullptr;
  void* merge_fn_ = nullptr;
  void* serialize_fn_ = nullptr;
  void* get_value_fn_ = nullptr;
  void* finalize_fn_ = nullptr;

  AggFn(const TExprNode& node, const SlotDescriptor& intermediate_slot_desc,
      const SlotDescriptor& output_slot_desc);

  /// Initializes the AggFn and its input expressions. May load the UDAF from LibCache
  /// if necessary.
  virtual Status Init(const RowDescriptor& desc, FragmentState* state) WARN_UNUSED_RESULT;
};

}

#endif
```

```cpp
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.

#include "exprs/agg-fn.h"

#include "codegen/llvm-codegen.h"
#include "exprs/anyval-util.h"
#include "exprs/scalar-expr.h"
#include "runtime/descriptors.h"
#include "runtime/fragment-state.h"
#include "runtime/lib-cache.h"

#include "common/names.h"

using namespace impala_udf;

namespace impala {

AggFn::AggFn(const TExprNode& tnode, const SlotDescriptor& intermediate_slot_desc,
    const SlotDescriptor& output_slot_desc)
  : Expr(tnode),
    is_merge_(tnode.agg_expr.is_merge_agg),
    intermediate_slot_desc_(intermediate_slot_desc),
    output_slot_desc_(output_slot_desc),
    arg_type_descs_(AnyValUtil::ColumnTypesToTypeDescs(
        ColumnType::FromThrift(tnode.agg_expr.arg_types))) {
  DCHECK(tnode.__isset.fn);
  DCHECK(tnode.fn.__isset.aggregate_fn);
  DCHECK_EQ(tnode.node_type, TExprNodeType::AGGREGATE_EXPR);
  DCHECK_EQ(ColumnType::FromThrift(tnode.type).type,
      ColumnType::FromThrift(fn_.ret_type).type);
  const string& fn_name = fn_.name.function_name;
  if (fn_name == "count") {
    agg_op_ = COUNT;
  } else if (fn_name == "min") {
    agg_op_ = MIN;
  } else if (fn_name == "max") {
    agg_op_ = MAX;
  } else if (fn_name == "sum" || fn_name == "sum_init_zero") {
    agg_op_ = SUM;
  } else if (fn_name == "avg") {
    agg_op_ = AVG;
  } else if (fn_name == "ndv" || fn_name == "ndv_no_finalize") {
    agg_op_ = NDV;
  } else {
    agg_op_ = OTHER;
  }
}

Status AggFn::Init(const RowDescriptor& row_desc, FragmentState* state) {
  // Initialize all children (i.e. input exprs to this aggregate expr).
  for (ScalarExpr* input_expr : children()) {
    RETURN_IF_ERROR(input_expr->Init(row_desc, /*is_entry_point*/ false, state));
  }

  // Initialize the aggregate expressions' internals.
  const TAggregateFunction& aggregate_fn = fn_.aggregate_fn;
  DCHECK_EQ(intermediate_slot_desc_.type().type,
      ColumnType::FromThrift(aggregate_fn.intermediate_type).type);
  DCHECK_EQ(output_slot_desc_.type().type, ColumnType::FromThrift(fn_.ret_type).type);

  time_t mtime = fn_.last_modified_time;
  // Load the function pointers. Must have init() and update().
  if (aggregate_fn.init_fn_symbol.empty() ||
      aggregate_fn.update_fn_symbol.empty() ||
      (aggregate_fn.merge_fn_symbol.empty() && !aggregate_fn.is_analytic_only_fn)) {
    // This path is only for partially implemented builtins.
    DCHECK_EQ(fn_.binary_type, TFunctionBinaryType::BUILTIN);
    stringstream ss;
    ss << "Function " << fn_.name.function_name << " is not implemented.";
    return Status(ss.str());
  }

  RETURN_IF_ERROR(LibCache::instance()->GetSoFunctionPtr(
      fn_.hdfs_location, aggregate_fn.init_fn_symbol, mtime, &init_fn_, &cache_entry_));
  RETURN_IF_ERROR(LibCache::instance()->GetSoFunctionPtr(fn_.hdfs_location,
      aggregate_fn.update_fn_symbol, mtime, &update_fn_, &cache_entry_));

  // Merge() is not defined for purely analytic function.
  if (!aggregate_fn.is_analytic_only_fn) {
    RETURN_IF_ERROR(LibCache::instance()->GetSoFunctionPtr(fn_.hdfs_location,
        aggregate_fn.merge_fn_symbol, mtime, &merge_fn_, &cache_entry_));
  }
  // Serialize(), GetValue(), Remove() and Finalize() are optional
  if (!aggregate_fn.serialize_fn_symbol.empty()) {
    RETURN_IF_ERROR(LibCache::instance()->GetSoFunctionPtr(fn_.hdfs_location,
        aggregate_fn.serialize_fn_symbol, mtime, &serialize_fn_, &cache_entry_));
  }
  if (!aggregate_fn.get_value_fn_symbol.empty()) {
    RETURN_IF_ERROR(LibCache::instance()->GetSoFunctionPtr(fn_.hdfs_location,
        aggregate_fn.get_value_fn_symbol, mtime, &get_value_fn_, &cache_entry_));
  }
  if (!aggregate_fn.remove_fn_symbol.empty()) {
    RETURN_IF_ERROR(LibCache::instance()->GetSoFunctionPtr(fn_.hdfs_location,
        aggregate_fn.remove_fn_symbol, mtime, &remove_fn_, &cache_entry_));
  }
  if (!aggregate_fn.finalize_fn_symbol.empty()) {
    RETURN_IF_ERROR(LibCache::instance()->GetSoFunctionPtr(fn_.hdfs_location,
        fn_.aggregate_fn.finalize_fn_symbol, mtime, &finalize_fn_, &cache_entry_));
  }
  return Status::OK();
}

Status AggFn::Create(const TExpr& texpr, const RowDescriptor& row_desc,
    const SlotDescriptor& intermediate_slot_desc, const SlotDescriptor& output_slot_desc,
    FragmentState* state, AggFn** agg_fn) {
  *agg_fn = nullptr;
  ObjectPool* pool = state->obj_pool();
  const TExprNode& texpr_node = texpr.nodes[0];
  DCHECK_EQ(texpr_node.node_type, TExprNodeType::AGGREGATE_EXPR);
  if (!texpr_node.__isset.fn) {
    return Status("Function not set in thrift AGGREGATE_EXPR node");
  }
  AggFn* new_agg_fn =
      pool->Add(new AggFn(texpr_node, intermediate_slot_desc, output_slot_desc));
  RETURN_IF_ERROR(Expr::CreateTree(texpr, pool, new_agg_fn));
  Status status = new_agg_fn->Init(row_desc, state);
  if (UNLIKELY(!status.ok())) {
    new_agg_fn->Close();
    return status;
  }
  for (ScalarExpr* input_expr : new_agg_fn->children()) {
    int fn_ctx_idx = 0;
    input_expr->AssignFnCtxIdx(&fn_ctx_idx);
  }
  *agg_fn = new_agg_fn;
  return Status::OK();
}

FunctionContext::TypeDesc AggFn::GetIntermediateTypeDesc() const {
  return AnyValUtil::ColumnTypeToTypeDesc(intermediate_slot_desc_.type());
}

FunctionContext::TypeDesc AggFn::GetOutputTypeDesc() const {
  return AnyValUtil::ColumnTypeToTypeDesc(output_slot_desc_.type());
}

Status AggFn::CodegenUpdateOrMergeFunction(
    LlvmCodeGen* codegen, llvm::Function** uda_fn) {
  const string& symbol =
      is_merge_ ? fn_.aggregate_fn.merge_fn_symbol : fn_.aggregate_fn.update_fn_symbol;
  vector<ColumnType> fn_arg_types;
  for (ScalarExpr* input_expr : children()) {
    fn_arg_types.push_back(input_expr->type());
  }
  // The intermediate value is passed as the last argument.
  fn_arg_types.push_back(intermediate_type());
  RETURN_IF_ERROR(codegen->LoadFunction(fn_, symbol, nullptr, fn_arg_types,
      fn_arg_types.size(), false, uda_fn, &cache_entry_));

  // Inline constants into the function body (if there is an IR body).
  if (!(*uda_fn)->isDeclaration()) {
    // TODO: IMPALA-4785: we should also replace references to GetIntermediateType()
    // with constants.
    codegen->InlineConstFnAttrs(GetOutputTypeDesc(), arg_type_descs_, *uda_fn);
    *uda_fn = codegen->FinalizeFunction(*uda_fn);
    if (*uda_fn == nullptr) {
      return Status(TErrorCode::UDF_VERIFY_FAILED, symbol, fn_.hdfs_location);
    }
  }
  return Status::OK();
}

void AggFn::Close() {
  // This also closes all the input expressions.
  Expr::Close();
}

void AggFn::Close(const vector<AggFn*>& exprs) {
  for (AggFn* expr : exprs) expr->Close();
}

string AggFn::DebugString() const {
  stringstream out;
  out << "AggFn(op=" << agg_op_;
  for (ScalarExpr* input_expr : children()) {
    out << " " << input_expr->DebugString() << ")";
  }
  out << ")";
  return out.str();
}

string AggFn::DebugString(const vector<AggFn*>& agg_fns) {
  stringstream out;
  out << "[";
  for (int i = 0; i < agg_fns.size(); ++i) {
    out << (i == 0 ? "" : " ") << agg_fns[i]->DebugString();
  }
  out << "]";
  return out.str();
}

}
```

```cpp
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.


#ifndef IMPALA_EXPRS_SCALAR_FN_CALL_H_
#define IMPALA_EXPRS_SCALAR_FN_CALL_H_

#include <string>

#include "exprs/scalar-expr.h"
#include "udf/udf.h"

namespace impala {

using impala_udf::FunctionContext;
using impala_udf::AnyVal;
using impala_udf::BooleanVal;
using impala_udf::TinyIntVal;
using impala_udf::SmallIntVal;
using impala_udf::IntVal;
using impala_udf::BigIntVal;
using impala_udf::FloatVal;
using impala_udf::DoubleVal;
using impala_udf::TimestampVal;
using impala_udf::StringVal;
using impala_udf::DecimalVal;
using impala_udf::DateVal;

class ScalarExprEvaluator;
class TExprNode;

/// Expr for evaluating a pre-compiled native or LLVM IR function that uses the UDF
/// interface (i.e. a scalar function). This class overrides GetCodegendComputeFnImpl() to
/// return a function that calls any child exprs and passes the results as arguments to
/// the specified scalar function.
///
/// If codegen is disabled, some native functions can be called without codegen, depending
/// on the native function's signature. However, since we can't write static code to call
/// every possible function signature, codegen may be required to generate the call to the
/// function even if codegen is disabled. Codegen will also be used for IR UDFs (note that
/// there is no way to specify both a native and IR library for a single UDF).
///
/// Scalar function call: An expr that returns a single scalar value and can be
/// implemented using the UDF interface. Note that this includes builtins, which although
/// not being user-defined still use the same interface as UDFs (i.e., they are
/// implemented as functions with signature "*Val (FunctionContext*, *Val, *Val...)").
///
/// TODO:
/// - Fix error reporting, e.g. reporting leaks
/// - Testing
///    - Test cancellation
///    - Type descs in UDA test harness
///    - Allow more functions to be NULL in UDA test harness
class ScalarFnCall : public ScalarExpr {
 public:
  virtual Status GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn)
      override WARN_UNUSED_RESULT;
  virtual std::string DebugString() const override;

 protected:
  friend class ScalarExpr;
  friend class ScalarExprEvaluator;

  virtual bool HasFnCtx() const override { return true; }

  ScalarFnCall(const TExprNode& node);
  virtual Status Init(const RowDescriptor& row_desc, bool is_entry_point,
      FragmentState* state) override WARN_UNUSED_RESULT;
  virtual Status OpenEvaluator(FunctionContext::FunctionStateScope scope,
      RuntimeState* state, ScalarExprEvaluator* eval) const override WARN_UNUSED_RESULT;
  virtual void CloseEvaluator(FunctionContext::FunctionStateScope scope,
      RuntimeState* state, ScalarExprEvaluator* eval) const override;
  virtual int ComputeVarArgsBufferSize() const override;
  /// Not all scalars functions are interpretable - see class comment.
  virtual bool IsInterpretable() const override;

  GENERATE_GET_VAL_INTERPRETED_OVERRIDES_FOR_ALL_SCALAR_TYPES

 private:
  /// If this function has var args, children()[vararg_start_idx_] is the first vararg
  /// argument.
  /// If this function does not have varargs, it is set to -1.
  const int vararg_start_idx_;

  /// Vector of all non-constant children expressions that need to be evaluated for
  /// each input row. The first element of each pair is the child expression and the
  /// second element in the value it must be evaluated into.
  std::vector<std::pair<Expr*, impala_udf::AnyVal*>> non_constant_children_;

  /// The UDF's prepare function, if specified. This is initialized in Prepare() and
  /// called in Open() (since we may have needed to codegen the function if it's from an
  /// IR module).
  CodegenFnPtr<impala_udf::UdfPrepare> prepare_fn_;

  /// THe UDF's close function, if specified. This is initialized in Prepare() and called
  /// in Close().
  CodegenFnPtr<impala_udf::UdfClose> close_fn_;

  /// A pointer to the function implementation, used by the interpreted code path. Set in
  /// Init() for BUILTIN and NATIVE functions. Not set for IR UDFs.
  void* scalar_fn_;

  /// Returns the number of non-vararg arguments
  int NumFixedArgs() const {
    return vararg_start_idx_ >= 0 ? vararg_start_idx_ : children_.size();
  }

  virtual int NumVarArgs() const { return children_.size() - NumFixedArgs(); }

  const ColumnType& VarArgsType() const {
    DCHECK_GE(NumVarArgs(), 1);
    return children_.back()->type();
  }

  /// Loads the native or IR function 'symbol' from HDFS and puts the result in *fn.
  /// If the function is loaded from an IR module, it cannot be called until the module
  /// has been JIT'd (i.e. after GetCodegendComputeFnImpl() has been called).
  Status GetFunction(LlvmCodeGen* codegen, const std::string& symbol,
      CodegenFnPtrBase* fn) WARN_UNUSED_RESULT;

  /// Loads the Prepare() and Close() functions for this ScalarFnCall. They could be
  /// native or IR functions. To load IR functions, the codegen object must have
  /// been created and any external LLVM module must have been linked already.
  Status LoadPrepareAndCloseFn(LlvmCodeGen* codegen) WARN_UNUSED_RESULT;

  /// Evaluates the non-constant children exprs. Used in the interpreted path.
  void EvaluateNonConstantChildren(
      ScalarExprEvaluator* eval, const TupleRow* row) const;

  /// Function to call scalar_fn_. Used in the interpreted path.
  template <typename RETURN_TYPE>
  RETURN_TYPE InterpretEval(ScalarExprEvaluator* eval, const TupleRow* row) const;
};
}

#endif
```

```cpp
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.

#include "exprs/scalar-fn-call.h"

#include <vector>
#include <gutil/strings/substitute.h>
#include <llvm/ExecutionEngine/ExecutionEngine.h>
#include <llvm/IR/Attributes.h>

#include <boost/preprocessor/punctuation/comma_if.hpp>
#include <boost/preprocessor/repetition/enum_params.hpp>
#include <boost/preprocessor/repetition/repeat.hpp>
#include <boost/preprocessor/repetition/repeat_from_to.hpp>

#include "codegen/codegen-anyval.h"
#include "codegen/llvm-codegen.h"
#include "exprs/anyval-util.h"
#include "exprs/scalar-expr-evaluator.h"
#include "runtime/fragment-state.h"
#include "runtime/hdfs-fs-cache.h"
#include "runtime/lib-cache.h"
#include "runtime/runtime-state.h"
#include "runtime/types.h"
#include "udf/udf-internal.h"
#include "util/debug-util.h"

#include "common/names.h"

using namespace impala;
using namespace impala_udf;
using namespace strings;
using std::move;
using std::pair;

// Maximum number of arguments the interpretation path supports.
#define MAX_INTERP_ARGS 20

ScalarFnCall::ScalarFnCall(const TExprNode& node)
  : ScalarExpr(node),
    vararg_start_idx_(node.__isset.vararg_start_idx ? node.vararg_start_idx : -1),
    prepare_fn_(),
    close_fn_(),
    scalar_fn_(NULL) {
  DCHECK_NE(fn_.binary_type, TFunctionBinaryType::JAVA);
}

Status ScalarFnCall::LoadPrepareAndCloseFn(LlvmCodeGen* codegen) {
  if (fn_.scalar_fn.__isset.prepare_fn_symbol) {
    RETURN_IF_ERROR(GetFunction(codegen, fn_.scalar_fn.prepare_fn_symbol,
        &prepare_fn_));
  }
  if (fn_.scalar_fn.__isset.close_fn_symbol) {
    RETURN_IF_ERROR(GetFunction(codegen, fn_.scalar_fn.close_fn_symbol,
        &close_fn_));
  }
  return Status::OK();
}

Status ScalarFnCall::Init(
    const RowDescriptor& desc, bool is_entry_point, FragmentState* state) {
  // Initialize children first.
  RETURN_IF_ERROR(ScalarExpr::Init(desc, is_entry_point, state));

  if (fn_.scalar_fn.symbol.empty()) {
    // This path is intended to only be used during development to test FE
    // code before the BE has implemented the function.
    // Having the failure in the BE (rather than during analysis) allows for
    // better FE testing.
    DCHECK_EQ(fn_.binary_type, TFunctionBinaryType::BUILTIN);
    stringstream ss;
    ss << "Function " << fn_.name.function_name << " is not implemented.";
    return Status(ss.str());
  }

  bool is_ir_udf = fn_.binary_type == TFunctionBinaryType::IR;
  if (!ShouldCodegen(state)) {
    // The interpreted code path must be handled in different ways depending on why
    // codegen was disabled. It may not be possible to evaluate the expr without
    // codegen or we may need to prepare the function for execution.
    if (is_ir_udf) {
      DCHECK(state->CodegenDisabledByQueryOption());
      return Status(Substitute("Cannot interpret LLVM IR UDF '$0': Codegen is needed. "
                               "Please set DISABLE_CODEGEN to false.",
          fn_.name.function_name));
    }

    // The templates for builtin or native UDFs used in the interpretation path
    // support up to MAX_INTERP_ARGS number of arguments only.
    if (NumFixedArgs() > MAX_INTERP_ARGS) {
      DCHECK_EQ(fn_.binary_type, TFunctionBinaryType::NATIVE);
      // CHAR or VARCHAR are not supported as input arguments or return values for UDFs.
      DCHECK(state->CodegenDisabledByQueryOption());
      return Status(Substitute(
          "Cannot interpret native UDF '$0': number of arguments is "
          "more than $1. Codegen is needed. Please set DISABLE_CODEGEN to false.",
          fn_.name.function_name, MAX_INTERP_ARGS));
    }
  }

  if (!is_ir_udf) {
    Status status = LibCache::instance()->GetSoFunctionPtr(fn_.hdfs_location,
        fn_.scalar_fn.symbol, fn_.last_modified_time, &scalar_fn_, &cache_entry_);
    if (!status.ok()) {
      if (fn_.binary_type == TFunctionBinaryType::BUILTIN) {
        // Builtins symbols should exist unless there is a version mismatch.
        return Status(TErrorCode::MISSING_BUILTIN, fn_.name.function_name,
            fn_.scalar_fn.symbol);
      } else {
        DCHECK_EQ(fn_.binary_type, TFunctionBinaryType::NATIVE);
        return Status(Substitute("Problem loading UDF '$0':\n$1",
            fn_.name.function_name, status.GetDetail()));
      }
    }
    // For IR UDF, the loading of the Init() and CloseContext() functions is deferred
    // until the first time GetCodegendComputeFn() is invoked.
    RETURN_IF_ERROR(LoadPrepareAndCloseFn(nullptr));
  }
  return Status::OK();
}

Status ScalarFnCall::OpenEvaluator(FunctionContext::FunctionStateScope scope,
    RuntimeState* state, ScalarExprEvaluator* eval) const {
  // Opens and inits children
  RETURN_IF_ERROR(ScalarExpr::OpenEvaluator(scope, state, eval));
  DCHECK_GE(fn_ctx_idx_, 0);
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);

  // Prepare staging_input_vals in case the interpreted evaluation path of
  // this function is invoked. staging_input_vals is preallocated here
  // so they can be reused across calls. If we have a codegen'd entry point
  // for this expression, allocating these input values may be unnecessary,
  // but they only add a small constant overhead on top of the ScalarExpr tree, so
  // we always allocate them for simplicity.
  vector<AnyVal*>* input_vals = fn_ctx->impl()->staging_input_vals();
  for (int i = 0; i < NumFixedArgs(); ++i) {
    AnyVal* input_val;
    RETURN_IF_ERROR(AllocateAnyVal(state, eval->expr_perm_pool(), children_[i]->type(),
        "Could not allocate expression value", &input_val));
    input_vals->push_back(input_val);
  }

  // Only evaluate constant arguments at the top level of function contexts.
  // If 'eval' was cloned, the constant values were copied from the parent.
  if (scope == FunctionContext::FRAGMENT_LOCAL) {
    vector<AnyVal*> constant_args;
    for (const ScalarExpr* child : children()) {
      AnyVal* const_val;
      RETURN_IF_ERROR(eval->GetConstValue(state, *child, &const_val));
      constant_args.push_back(const_val);
    }
    fn_ctx->impl()->SetConstantArgs(move(constant_args));

    // If we're calling MathFunctions::RoundUpTo(), we need to set output_scale_
    // which determines how many decimal places are printed.
    // TODO: Move this to Expr initialization when IMPALA-4743 is fixed.
    if (this == &eval->root() &&
        fn_.name.function_name == "round" &&
        type_.type == TYPE_DOUBLE &&
        children_.size() == 2) {
      BigIntVal* scale_arg = reinterpret_cast<BigIntVal*>(constant_args[1]);
      if (scale_arg != nullptr) eval->output_scale_ = scale_arg->val;
    }
  }

  // Now we have the constant values, cache them so that the interpreted path can
  // call the UDF without reevaluating the arguments. 'staging_input_vals' and
  // 'varargs_buffer' in the FunctionContext are used to pass fixed and variable-length
  // arguments respectively. 'non_constant_args()' in the FunctionContext will contain
  // pointers to the remaining (non-constant) children that are evaluated for every row.
  vector<pair<ScalarExpr*, AnyVal*>> non_constant_args;
  uint8_t* varargs_buffer = fn_ctx->impl()->varargs_buffer();
  for (int i = 0; i < children_.size(); ++i) {
    AnyVal* input_arg;
    int arg_bytes = AnyValUtil::AnyValSize(children_[i]->type());
    if (i < NumFixedArgs()) {
      input_arg = (*fn_ctx->impl()->staging_input_vals())[i];
    } else {
      input_arg = reinterpret_cast<AnyVal*>(varargs_buffer);
      varargs_buffer += arg_bytes;
    }
    // IMPALA-4586: Cache constant arguments only if the frontend has rewritten them
    // into literal expressions. This gives the frontend control over how expressions
    // are evaluated. This means that setting enable_expr_rewrites=false will also
    // disable caching of non-literal constant expressions, which gives the old
    // behaviour (before this caching optimisation was added) of repeatedly evaluating
    // exprs that are constant according to is_constant(). For exprs that are not truly
    // constant (yet is_constant() returns true for) e.g. non-deterministic UDFs, this
    // means that setting enable_expr_rewrites=false works as a safety valve to get
    // back the old behaviour, before constant expr folding or caching was added.
    // TODO: once we can annotate UDFs as non-deterministic (IMPALA-4606), we should
    // be able to trust is_constant() and switch back to that.
    if (children_[i]->IsLiteral()) {
      const AnyVal* constant_arg = fn_ctx->impl()->constant_args()[i];
      DCHECK(constant_arg != nullptr);
      memcpy(input_arg, constant_arg, arg_bytes);
    } else {
      non_constant_args.emplace_back(children_[i], input_arg);
    }
  }
  fn_ctx->impl()->SetNonConstantArgs(move(non_constant_args));

  const impala_udf::UdfPrepare prepare_fn = prepare_fn_.load();
  if (prepare_fn != nullptr) {
    if (scope == FunctionContext::FRAGMENT_LOCAL) {
      prepare_fn(fn_ctx, FunctionContext::FRAGMENT_LOCAL);
      if (fn_ctx->has_error()) return Status(fn_ctx->error_msg());
    }
    prepare_fn(fn_ctx, FunctionContext::THREAD_LOCAL);
    if (fn_ctx->has_error()) return Status(fn_ctx->error_msg());
  }

  return Status::OK();
}

void ScalarFnCall::CloseEvaluator(FunctionContext::FunctionStateScope scope,
    RuntimeState* state, ScalarExprEvaluator* eval) const {
  DCHECK_GE(fn_ctx_idx_, 0);
  const impala_udf::UdfClose close_fn = close_fn_.load();
  if (close_fn != NULL) {
    FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
    close_fn(fn_ctx, FunctionContext::THREAD_LOCAL);
    if (scope == FunctionContext::FRAGMENT_LOCAL) {
      close_fn(fn_ctx, FunctionContext::FRAGMENT_LOCAL);
    }
  }
  ScalarExpr::CloseEvaluator(scope, state, eval);
}

// Dynamically loads the pre-compiled UDF and codegens a function that calls each child's
// codegen'd function, then passes those values to the UDF and returns the result.
// Example generated IR for a UDF with signature
//    create function Udf(double, int...) returns double
//    select Udf(1.0, 2, 3, 4, 5)
// define { i8, double } @UdfWrapper(i8* %context, %"class.impala::TupleRow"* %row) {
// entry:
//   %arg_val = call { i8, double }
//      @ExprWrapper(i8* %context, %"class.impala::TupleRow"* %row)
//   %arg_ptr = alloca { i8, double }
//   store { i8, double } %arg_val, { i8, double }* %arg_ptr
//   %arg_val1 = call i64 @ExprWrapper1(i8* %context, %"class.impala::TupleRow"* %row)
//   store i64 %arg_val1, i64* inttoptr (i64 89111072 to i64*)
//   %arg_val2 = call i64 @ExprWrapper2(i8* %context, %"class.impala::TupleRow"* %row)
//   store i64 %arg_val2, i64* inttoptr (i64 89111080 to i64*)
//   %arg_val3 = call i64 @ExprWrapper3(i8* %context, %"class.impala::TupleRow"* %row)
//   store i64 %arg_val3, i64* inttoptr (i64 89111088 to i64*)
//   %arg_val4 = call i64 @ExprWrapper4(i8* %context, %"class.impala::TupleRow"* %row)
//   store i64 %arg_val4, i64* inttoptr (i64 89111096 to i64*)
//   %result = call { i8, double }
//      @_Z14VarSumMultiplyPN10impala_udf15FunctionContextERKNS_9DoubleValEiPKNS_6IntValE(
//        %"class.impala_udf::FunctionContext"* inttoptr
//            (i64 37522464 to %"class.impala_udf::FunctionContext"*),
//        {i8, double }* %arg_ptr,
//        i32 4,
//        i64* inttoptr (i64 89111072 to i64*))
//   ret { i8, double } %result
Status ScalarFnCall::GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) {
  vector<ColumnType> arg_types;
  for (ScalarExpr* child : children_) arg_types.push_back(child->type());
  llvm::Function* udf;
  RETURN_IF_ERROR(codegen->LoadFunction(fn_, fn_.scalar_fn.symbol, &type_, arg_types,
      NumFixedArgs(), vararg_start_idx_ != -1, &udf, &cache_entry_));
  // Inline constants into the function if it has an IR body.
  if (!udf->isDeclaration()) {
    codegen->InlineConstFnAttrs(AnyValUtil::ColumnTypeToTypeDesc(type_),
        AnyValUtil::ColumnTypesToTypeDescs(arg_types), udf);
    udf = codegen->FinalizeFunction(udf);
    if (udf == NULL) {
      return Status(
          TErrorCode::UDF_VERIFY_FAILED, fn_.scalar_fn.symbol, fn_.hdfs_location);
    }
  }

  if (fn_.binary_type == TFunctionBinaryType::IR) {
    // LoadFunction() should have linked the IR module into 'codegen'. Now load the
    // Prepare() and Close() functions from 'codegen'.
    RETURN_IF_ERROR(LoadPrepareAndCloseFn(codegen));
  }

  // Create wrapper that computes args and calls UDF
  stringstream fn_name;
  fn_name << udf->getName().str() << "Wrapper";

  llvm::Value* args[2];
  *fn = CreateIrFunctionPrototype(fn_name.str(), codegen, &args);
  llvm::Value* eval = args[0];
  llvm::Value* row = args[1];
  llvm::BasicBlock* block = llvm::BasicBlock::Create(codegen->context(), "entry", *fn);
  LlvmBuilder builder(block);

  // Populate UDF arguments
  vector<llvm::Value*> udf_args;

  // First argument is always FunctionContext*.
  // Index into our registered offset in the ScalarFnEvaluator.
  llvm::Value* eval_gep = builder.CreateStructGEP(NULL, eval, 1, "eval_gep");
  llvm::Value* fn_ctxs_base = builder.CreateLoad(eval_gep, "fn_ctxs_base");
  // Use GEP to add our index to the base pointer
  llvm::Value* fn_ctx_ptr =
      builder.CreateConstGEP1_32(fn_ctxs_base, fn_ctx_idx_, "fn_ctx_ptr");
  llvm::Value* fn_ctx = builder.CreateLoad(fn_ctx_ptr, "fn_ctx");
  udf_args.push_back(fn_ctx);

  // Allocate a varargs array. The array's entry type is the appropriate AnyVal subclass.
  // E.g. if the vararg type is STRING, and the function is called with 10 arguments, we
  // allocate a StringVal[10] array. We allocate the buffer with Alloca so that LLVM can
  // optimise out the buffer once the function call is inlined.
  llvm::Value* varargs_buffer = NULL;
  if (vararg_start_idx_ != -1) {
    llvm::Type* unlowered_varargs_type =
        CodegenAnyVal::GetUnloweredType(codegen, VarArgsType());
    varargs_buffer = codegen->CreateEntryBlockAlloca(builder, unlowered_varargs_type,
        NumVarArgs(), FunctionContextImpl::VARARGS_BUFFER_ALIGNMENT, "varargs_buffer");
  }

  // Call children to populate remaining arguments
  for (int i = 0; i < GetNumChildren(); ++i) {
    llvm::Function* child_fn = NULL;
    vector<llvm::Value*> child_fn_args;
    // Set 'child_fn' to the codegen'd function, sets child_fn == NULL if codegen fails
    RETURN_IF_ERROR(children_[i]->GetCodegendComputeFn(codegen, false, &child_fn));
    child_fn_args.push_back(eval);
    child_fn_args.push_back(row);

    // Call 'child_fn', adding the result to either 'udf_args' or 'varargs_buffer'
    DCHECK(child_fn != NULL);
    llvm::Type* arg_type = CodegenAnyVal::GetUnloweredType(codegen, children_[i]->type());
    llvm::Value* arg_val_ptr;
#ifdef __aarch64__
    PrimitiveType col_type = children_[i]->type().type;
#endif
    if (i < NumFixedArgs()) {
#ifndef __aarch64__
      // Allocate space to store 'child_fn's result so we can pass the pointer to the UDF.
      arg_val_ptr = codegen->CreateEntryBlockAlloca(builder, arg_type, "arg_val_ptr");
      udf_args.push_back(arg_val_ptr);
#else
      if (col_type != TYPE_BOOLEAN and col_type != TYPE_TINYINT
          and col_type != TYPE_SMALLINT) {
        arg_val_ptr = codegen->CreateEntryBlockAlloca(builder, arg_type, "arg_val_ptr");
        udf_args.push_back(arg_val_ptr);
      }
#endif
    } else {
      // Store the result of 'child_fn' in varargs_buffer[i].
      arg_val_ptr =
          builder.CreateConstGEP1_32(varargs_buffer, i - NumFixedArgs(), "arg_val_ptr");
    }
#ifndef __aarch64__
    DCHECK_EQ(arg_val_ptr->getType(), arg_type->getPointerTo());
    // The result of the call must be stored in a lowered AnyVal
    llvm::Value* lowered_arg_val_ptr = builder.CreateBitCast(arg_val_ptr,
        CodegenAnyVal::GetLoweredPtrType(codegen, children_[i]->type()),
        "lowered_arg_val_ptr");
#else
    llvm::Value* lowered_arg_val_ptr;
    if (col_type == TYPE_BOOLEAN or col_type == TYPE_TINYINT
        or col_type == TYPE_SMALLINT) {
      lowered_arg_val_ptr = codegen->CreateEntryBlockAlloca(builder,
          CodegenAnyVal::GetLoweredType(codegen, children_[i]->type()), 1,
          FunctionContextImpl::VARARGS_BUFFER_ALIGNMENT, "lowered_arg_val_ptr");
    } else {
      lowered_arg_val_ptr = builder.CreateBitCast(arg_val_ptr,
          CodegenAnyVal::GetLoweredPtrType(codegen, children_[i]->type()),
          "lowered_arg_val_ptr");
    }
#endif
    CodegenAnyVal::CreateCall(
        codegen, &builder, child_fn, child_fn_args, "arg_val", lowered_arg_val_ptr);
#ifdef __aarch64__
    if (col_type == TYPE_BOOLEAN or col_type == TYPE_TINYINT
        or col_type == TYPE_SMALLINT) {
      if (i < NumFixedArgs()) {
        arg_val_ptr = builder.CreateTruncOrBitCast(lowered_arg_val_ptr,
            CodegenAnyVal::GetUnloweredPtrType(codegen, children_[i]->type()),
            "arg_val_ptr");
        udf_args.push_back(arg_val_ptr);
      } else {
        llvm::Value* tmp_ptr = builder.CreateTruncOrBitCast(lowered_arg_val_ptr,
            CodegenAnyVal::GetUnloweredPtrType(codegen, children_[i]->type()),
            "tmp_ptr");
        builder.CreateStore(builder.CreateLoad(tmp_ptr), arg_val_ptr);
      }
    }
#endif
  }

  if (vararg_start_idx_ != -1) {
    // We've added the FunctionContext argument plus any non-variadic arguments
    DCHECK_EQ(udf_args.size(), vararg_start_idx_ + 1);
    DCHECK_GE(GetNumChildren(), 1);
    // Add the number of varargs
    udf_args.push_back(codegen->GetI32Constant(NumVarArgs()));
    // Add all the accumulated vararg inputs as one input argument.
    llvm::PointerType* vararg_type =
        CodegenAnyVal::GetUnloweredPtrType(codegen, VarArgsType());
    udf_args.push_back(builder.CreateBitCast(varargs_buffer, vararg_type, "varargs"));
  }

  // Call UDF
  llvm::Value* result_val =
      CodegenAnyVal::CreateCall(codegen, &builder, udf, udf_args, "result");
  builder.CreateRet(result_val);

  *fn = codegen->FinalizeFunction(*fn);
  if (*fn == NULL) {
    return Status(
        TErrorCode::UDF_VERIFY_FAILED, fn_.scalar_fn.symbol, fn_.hdfs_location);
  }
  return Status::OK();
}

Status ScalarFnCall::GetFunction(LlvmCodeGen* codegen, const string& symbol,
    CodegenFnPtrBase* fn) {
  if (fn_.binary_type == TFunctionBinaryType::NATIVE
      || fn_.binary_type == TFunctionBinaryType::BUILTIN) {
    void* raw_fn;
    const Status status = LibCache::instance()->GetSoFunctionPtr(
        fn_.hdfs_location, symbol, fn_.last_modified_time, &raw_fn, &cache_entry_);
    fn->store(raw_fn);
    return status;
  } else {
    DCHECK_EQ(fn_.binary_type, TFunctionBinaryType::IR);
    DCHECK(codegen != NULL);
    llvm::Function* ir_fn = codegen->GetFunction(symbol, false);
    if (ir_fn == NULL) {
      stringstream ss;
      ss << "Unable to locate function " << symbol << " from LLVM module "
         << fn_.hdfs_location;
      return Status(ss.str());
    }
    ir_fn = codegen->FinalizeFunction(ir_fn);
    codegen->AddFunctionToJit(ir_fn, fn);
    return Status::OK();
  }
}

void ScalarFnCall::EvaluateNonConstantChildren(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  for (pair<ScalarExpr*, AnyVal*> child : fn_ctx->impl()->non_constant_args()) {
    void* val = eval->GetValue(*(child.first), row);
    AnyValUtil::SetAnyVal(val, child.first->type(), child.second);
  }
}

template<typename RETURN_TYPE>
RETURN_TYPE ScalarFnCall::InterpretEval(ScalarExprEvaluator* eval,
    const TupleRow* row) const {
  DCHECK(scalar_fn_ != NULL) << DebugString();
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  vector<AnyVal*>* input_vals = fn_ctx->impl()->staging_input_vals();
  EvaluateNonConstantChildren(eval, row);

  if (vararg_start_idx_ == -1) {
    switch (children_.size()) {
#define ARG_DECL_ONE(z, n, data) BOOST_PP_COMMA_IF(n) const AnyVal&
#define ARG_DECL_LIST(n) \
  FunctionContext* BOOST_PP_COMMA_IF(n) BOOST_PP_REPEAT(n, ARG_DECL_ONE, unused)
#define ARG_ONE(z, n, data) BOOST_PP_COMMA_IF(n) *(*input_vals)[n]
#define ARG_LIST(n) fn_ctx BOOST_PP_COMMA_IF(n) BOOST_PP_REPEAT(n, ARG_ONE, unused)

   // Expands to code snippet like the following for X from 0 to 20:
   // case X:
   //     typedef RETURN_TYPE (*ScalarFnX)(FunctionContext*, const AnyVal& a1, ...,
   //         const AnyVal& aX);
   //     return reinterpret_cast<ScalarFnX>(scalar_fn_)(fn_ctx, *(*input_vals)[0], ...,
   //         *(*input_vals)[X-1]);
#define SCALAR_FN_TYPE(n) BOOST_PP_CAT(ScalarFn, n)
#define INTERP_SCALAR_FN(z, n, unused)                                       \
      case n:                                                                \
        typedef RETURN_TYPE (*SCALAR_FN_TYPE(n))(ARG_DECL_LIST(n));          \
        return reinterpret_cast<SCALAR_FN_TYPE(n)>(scalar_fn_)(ARG_LIST(n));

      // Support up to MAX_INTERP_ARGS arguments in the interpretation path
      BOOST_PP_REPEAT_FROM_TO(0, BOOST_PP_ADD(MAX_INTERP_ARGS, 1),
          INTERP_SCALAR_FN, unused)

      default:
        DCHECK(false) << "Interpreted path not implemented.";
    }
  } else {
    int num_varargs = children_.size() - NumFixedArgs();
    const AnyVal* varargs = reinterpret_cast<AnyVal*>(fn_ctx->impl()->varargs_buffer());
    switch (NumFixedArgs()) {

   // Expands to code snippet like the following for X from 0 to 20:
   // case X:
   //     typedef RETURN_TYPE (*VarargFnX)(FunctionContext*, const AnyVal& a1, ...,
   //         const AnyVal& aX, int num_varargs, const AnyVal* varargs);
   //     return reinterpret_cast<VarargFnX>(scalar_fn_)(fn_ctx, *(*input_vals)[0], ...,
   //         *(*input_vals)[X-1], num_varargs, varargs);
#define SCALAR_VARARG_FN_TYPE(n) BOOST_PP_CAT(VarargFn, n)
#define INTERP_SCALAR_VARARG_FN(z, n, text)                                        \
      case n:                                                                      \
        typedef RETURN_TYPE (*SCALAR_VARARG_FN_TYPE(n))(ARG_DECL_LIST(n), int,     \
            const AnyVal*);                                                        \
        return reinterpret_cast<SCALAR_VARARG_FN_TYPE(n)>(scalar_fn_)(ARG_LIST(n), \
            num_varargs, varargs);

      BOOST_PP_REPEAT_FROM_TO(0, BOOST_PP_ADD(MAX_INTERP_ARGS, 1),
         INTERP_SCALAR_VARARG_FN, unused)

      default:
        DCHECK(false) << "Interpreted path not implemented.";
    }
  }
  return RETURN_TYPE::null();
}

// Macro to generate implementations for the below functions. 'val_type' is
// a UDF type name, e.g. IntVal and 'type_validation' is a DCHECK expression
// referencing 'type_' to assert that the function is only called on expressions
// of the appropriate type.
// * ScalarFnCall::GetBooleanValInterpreted()
// * ScalarFnCall::GetTinyIntValInterpreted()
// * ScalarFnCall::GetSmallIntValInterpreted()
// * ScalarFnCall::GetIntValInterpreted()
// * ScalarFnCall::GetBigIntValInterpreted()
// * ScalarFnCall::GetFloatValInterpreted()
// * ScalarFnCall::GetDoubleValInterpreted()
// * ScalarFnCall::GetStringValInterpreted()
// * ScalarFnCall::GetTimestampValInterpreted()
// * ScalarFnCall::GetDecimalValInterpreted()
// * ScalarFnCall::GetDateValInterpreted()
#pragma push_macro("GET_VAL_INTERPRETED")
#define GET_VAL_INTERPRETED(val_type, type_validation)        \
  val_type ScalarFnCall::Get##val_type##Interpreted(          \
      ScalarExprEvaluator* eval, const TupleRow* row) const { \
    DCHECK(type_validation) << type_.DebugString();           \
    DCHECK(eval != nullptr);                                  \
    return InterpretEval<val_type>(eval, row);                \
  }

GET_VAL_INTERPRETED(BooleanVal, type_.type == PrimitiveType::TYPE_BOOLEAN);
GET_VAL_INTERPRETED(TinyIntVal, type_.type == PrimitiveType::TYPE_TINYINT);
GET_VAL_INTERPRETED(SmallIntVal, type_.type == PrimitiveType::TYPE_SMALLINT);
GET_VAL_INTERPRETED(IntVal, type_.type == PrimitiveType::TYPE_INT);
GET_VAL_INTERPRETED(BigIntVal, type_.type == PrimitiveType::TYPE_BIGINT);
GET_VAL_INTERPRETED(FloatVal, type_.type == PrimitiveType::TYPE_FLOAT);
GET_VAL_INTERPRETED(DoubleVal, type_.type == PrimitiveType::TYPE_DOUBLE);
GET_VAL_INTERPRETED(StringVal,
    type_.IsStringType() || type_.type == PrimitiveType::TYPE_FIXED_UDA_INTERMEDIATE);
GET_VAL_INTERPRETED(TimestampVal, type_.type == PrimitiveType::TYPE_TIMESTAMP);
GET_VAL_INTERPRETED(DecimalVal, type_.type == PrimitiveType::TYPE_DECIMAL);
GET_VAL_INTERPRETED(DateVal, type_.type == PrimitiveType::TYPE_DATE);
#pragma pop_macro("GET_VAL_INTERPRETED")

string ScalarFnCall::DebugString() const {
  stringstream out;
  out << "ScalarFnCall(udf_type=" << fn_.binary_type << " location=" << fn_.hdfs_location
      << " symbol_name=" << fn_.scalar_fn.symbol <<  ScalarExpr::DebugString() << ")";
  return out.str();
}

bool ScalarFnCall::IsInterpretable() const {
  return fn_.binary_type != TFunctionBinaryType::IR && NumFixedArgs() <= MAX_INTERP_ARGS;
}

int ScalarFnCall::ComputeVarArgsBufferSize() const {
  for (int i = NumFixedArgs(); i < children_.size(); ++i) {
    // All varargs should have same type.
    DCHECK_EQ(children_[i]->type(), VarArgsType());
  }
  return NumVarArgs() == 0 ? 0 : NumVarArgs() * AnyValUtil::AnyValSize(VarArgsType());
}
```

请你继续回答：Apache Impala 的符号查找机制（symbol lookup）主要在后端（Backend，C++）实现，用于运行时动态加载内置函数（builtin functions）的实现，尤其支持 LLVM 代码生成（codegen）路径。

是的，你提供的这些源码片段正是 Apache Impala **符号查找机制（symbol lookup）** 的核心实现部分。我来结合这些代码，完整、清晰地解释 Impala 是**如何在后端动态查找和加载内置函数符号**的。

### 核心机制概览

Impala 的内置标量函数（scalar builtin functions）主要通过两种方式被调用：

1. **解释执行路径（interpreted path）**：直接调用 C++ 函数指针（通过 `dlsym()` 从 `.so` 加载）。
2. **代码生成路径（codegen path）**：使用 LLVM JIT 编译，函数以 **LLVM IR 函数** 的形式存在于预编译的 `.bc` 模块中。

**符号查找机制** 主要服务于 **第2种路径** —— **LLVM codegen**，让前端（Java）只需要提供**人类可读的函数名**（如 `impala::MathFunctions::Abs`），而不需要硬编码 mangled 符号。

### 关键实现位置与流程

#### 1. 前端注册阶段（impala_functions.py → ScalarBuiltins.java）

- `impala_functions.py` 中的 `<backend symbol>` 字段现在大多是**可读形式**，如：
  ```python
  'impala::MathFunctions::Abs'
  ```
- `gen_builtins_catalog.py` 生成的 `ScalarBuiltins.java` 中调用：
  ```java
  db.addScalarBuiltin("abs", "impala::MathFunctions::Abs", ...);
  ```

#### 2. 后端加载函数：`LlvmCodeGen::LoadFunction()`

这是**符号查找的核心入口**（你提供的 `llvm-codegen.h/cc` 中）：

```cpp
Status LlvmCodeGen::LoadFunction(const TFunction& fn, const std::string& symbol,
    ..., llvm::Function** llvm_fn, LibCacheEntry** cache_entry) {
  if (fn.binary_type == TFunctionBinaryType::NATIVE || 
      (fn.binary_type == TFunctionBinaryType::BUILTIN && broken_builtin)) {
    // 旧路径：通过 LibCache + dlsym() 加载 native 函数指针
    void* fn_ptr = ...;
    execution_engine()->addGlobalMapping(*llvm_fn, fn_ptr);
  } else if (fn.binary_type == TFunctionBinaryType::BUILTIN) {
    // 新路径：从预编译的 IR 模块中查找函数（关键！）
    *llvm_fn = GetFunction(symbol, true);  // symbol 是可读名，如 "impala::MathFunctions::Abs"
  } else {
    // IR UDF：链接外部 .bc 模块，然后查找
    RETURN_IF_ERROR(LinkModuleFromHdfs(...));
    *llvm_fn = GetFunction(symbol, true);
  }
}
```

#### 3. 实际查找函数：`LlvmCodeGen::GetFunction(const string& symbol, bool clone)`

```cpp
llvm::Function* LlvmCodeGen::GetFunction(const string& symbol, bool clone) {
  llvm::Function* fn = module_->getFunction(symbol.c_str());  // 直接用名字查找！
  if (fn == NULL) return NULL;

  RETURN_IF_ERROR(MaterializeFunction(fn));  // 懒加载：解析 bitcode，填充函数体
  if (clone) return CloneFunction(fn);
  return fn;
}
```

- 这里 **不再依赖 mangled 名**，直接用 `module_->getFunction("impala::MathFunctions::Abs")` 查找。
- `MaterializeFunction()` 会递归物化（materialize）该函数及其所有被调用函数的 IR 体。

#### 4. 预编译 IR 模块的加载（`CreateImpalaCodegen` → `LoadModuleFromMemory`）

- 启动时，Impala 会加载一个**预编译的 LLVM bitcode 模块**（`impala_llvm_*.bc`），里面包含所有内置函数的 IR 实现（在 `be/src/exprs/*-ir.cc` 中定义）。
- 这个模块中的函数名就是人类可读的 `impala::XXX::FunctionName`。

#### 5. JIT 编译与调用

- 在 `FinalizeModule()` 中，LLVM 会优化并编译所有被 `AddFunctionToJit()` 注册的函数。
- 最终生成的机器码函数指针通过 `execution_engine()->getPointerToFunction(fn)` 获取。

### 为什么这样设计？优势总结

| 优势 | 说明 |
|------|------|
| **避免 mangled 名维护** | 不再需要用 `nm` 手动提取 `_ZN6impala...` 这种符号，减少出错。 |
| **跨平台兼容** | mangled 名依赖 ABI 和编译器版本，可读名更稳定。 |
| **支持内联优化** | 所有函数都在同一个 LLVM module 中，LLVM 可以自由内联 handcrafted 和 cross-compiled 函数。 |
| **懒加载** | 函数只有在真正被使用时才解析 bitcode（materialize），节省内存和启动时间。 |
| **统一接口** | 无论是 builtin 还是 IR UDF，都通过同一套机制加载。 |

### 总结：Impala 的符号查找机制到底是怎么实现的？

> Impala 在后端维护了一个**预编译的 LLVM IR 模块**，其中所有内置函数都以**人类可读的 C++ 名**（如 `impala::TimestampFunctions::Now`）存在。  
> 当需要调用某个内置函数时，后端通过 `module_->getFunction("impala::TimestampFunctions::Now")` **直接按名字查找**，然后懒加载其 IR 实现，最后通过 LLVM JIT 编译成本地机器码执行。

这正是你之前看到的从“硬编码 mangled 符号”到“动态符号查找”的演进结果 —— 现在开发者新增内置函数时，只需：

1. 在 C++ 中实现 `impala::XXX::MyFunction`
2. 在 `impala_functions.py` 中写 `'impala::XXX::MyFunction'`
3. 重新编译即可，无需再跑 `nm` 提取符号。

这就是 Apache Impala 现代 builtin 函数系统的核心机制。
****************************************************************************************
请仔细解释下：
C++

```
Status LlvmCodeGen::LoadFunction(const TFunction& fn, const string& symbol,
    const ColumnType* return_type, const vector<ColumnType>& arg_types,
    int num_fixed_args, bool has_varargs, llvm::Function** llvm_fn,
    LibCacheEntry** cache_entry) {
  DCHECK_GE(arg_types.size(), num_fixed_args);
  DCHECK(has_varargs || arg_types.size() == num_fixed_args);
  DCHECK(!has_varargs || arg_types.size() > num_fixed_args);
  // from_utc_timestamp() and to_utc_timestamp() have inline ASM that cannot be JIT'd.
  // TimestampFunctions::AddSub() contains a try/catch which doesn't work in JIT'd
  // code. Always use the interpreted version of these functions.
  // TODO: fix these built-in functions so we don't need 'broken_builtin' below.
  bool broken_builtin = fn.name.function_name == "from_utc_timestamp"
      || fn.name.function_name == "to_utc_timestamp"
      || symbol.find("AddSub") != string::npos;
  if (fn.binary_type == TFunctionBinaryType::NATIVE
      || (fn.binary_type == TFunctionBinaryType::BUILTIN && broken_builtin)) {
    // In this path, we are calling a precompiled native function, either a UDF
    // in a .so or a builtin using the UDF interface.
    void* fn_ptr;
    Status status = LibCache::instance()->GetSoFunctionPtr(
        fn.hdfs_location, symbol, fn.last_modified_time, &fn_ptr, cache_entry);
    if (!status.ok() && fn.binary_type == TFunctionBinaryType::BUILTIN) {
      // Builtins symbols should exist unless there is a version mismatch.
      status.AddDetail(
          ErrorMsg(TErrorCode::MISSING_BUILTIN, fn.name.function_name, symbol).msg());
    }
    RETURN_IF_ERROR(status);
    DCHECK(fn_ptr != NULL);

    // Per the x64 ABI, DecimalVals are returned via a DecimalVal* output argument.
    // So, the return type is void.
    bool is_decimal = return_type != NULL && return_type->type == TYPE_DECIMAL;
    llvm::Type* llvm_return_type = return_type == NULL || is_decimal ?
        void_type() :
        CodegenAnyVal::GetLoweredType(this, *return_type);

    // Convert UDF function pointer to Function*. Start by creating a function
    // prototype for it.
    FnPrototype prototype(this, symbol, llvm_return_type);

    if (is_decimal) {
      // Per the x64 ABI, DecimalVals are returned via a DecmialVal* output argument
      llvm::Type* output_type = CodegenAnyVal::GetUnloweredPtrType(this, *return_type);
      prototype.AddArgument("output", output_type);
    }

    // The "FunctionContext*" argument.
    prototype.AddArgument("ctx", GetNamedPtrType("class.impala_udf::FunctionContext"));

    // The "fixed" arguments for the UDF function, followed by the variable arguments,
    // if any.
    for (int i = 0; i < num_fixed_args; ++i) {
      llvm::Type* arg_type = CodegenAnyVal::GetUnloweredPtrType(this, arg_types[i]);
      prototype.AddArgument(Substitute("fixed_arg_$0", i), arg_type);
    }

    if (has_varargs) {
      prototype.AddArgument("num_var_arg", i32_type());
      // Get the vararg type from the first vararg.
      prototype.AddArgument(
          "var_arg", CodegenAnyVal::GetUnloweredPtrType(this, arg_types[num_fixed_args]));
    }

    // Create a Function* with the generated type. This is only a function
    // declaration, not a definition, since we do not create any basic blocks or
    // instructions in it.
    *llvm_fn = prototype.GeneratePrototype(nullptr, nullptr);
#ifdef __aarch64__
    if (is_decimal) {
      // Mark first argument as sret
      (*llvm_fn)->addAttribute(1, llvm::Attribute::StructRet);
    }
#endif
    // Associate the dynamically loaded function pointer with the Function* we defined.
    // This tells LLVM where the compiled function definition is located in memory.
    execution_engine()->addGlobalMapping(*llvm_fn, fn_ptr);
    // Disable the codegen cache because codegen cache uses the llvm module bitcode as
    // the key while the bitcode doesn't contain the global function mapping of the
    // execution engine. If the mapping is changed during running, like udf recreation,
    // the function mapping in the cache could point to an old address and lead to a
    // crash while calling the udf,  so block the codegen cache for native udfs.
    // Builtin functions should not have the issue, because they should not change
    // during runtime.
    if (fn.binary_type == TFunctionBinaryType::NATIVE) {
      // Should be before compilation.
      DCHECK(!is_compiled_);
      codegen_cache_enabled_ = false;
    }
  } else if (fn.binary_type == TFunctionBinaryType::BUILTIN) {
    // In this path, we're running a builtin with the UDF interface. The IR is
    // in the llvm module. Builtin functions may use Expr::GetConstant(). Clone the
    // function so that we can replace constants in the copied function.
    *llvm_fn = GetFunction(symbol, true);
    if (*llvm_fn == NULL) {
      // Builtins symbols should exist unless there is a version mismatch.
      return Status(Substitute("Builtin '$0' with symbol '$1' does not exist. Verify "
                               "that all your impalads are the same version.",
          fn.name.function_name, symbol));
    }
    // Rename the function to something more readable than the mangled name.
    string demangled_name = SymbolsUtil::DemangleNoArgs((*llvm_fn)->getName().str());
    (*llvm_fn)->setName(demangled_name);
  } else {
    // We're running an IR UDF.
    DCHECK_EQ(fn.binary_type, TFunctionBinaryType::IR);

    // Link the UDF module into this query's main module so the UDF's functions are
    // available in the main module.
    RETURN_IF_ERROR(LinkModuleFromHdfs(fn.hdfs_location, fn.last_modified_time));

    *llvm_fn = GetFunction(symbol, true);
    if (*llvm_fn == NULL) {
      return Status(Substitute("Unable to load function '$0' from LLVM module '$1'",
          symbol, fn.hdfs_location));
    }
    // Rename the function to something more readable than the mangled name.
    string demangled_name = SymbolsUtil::DemangleNoArgs((*llvm_fn)->getName().str());
    (*llvm_fn)->setName(demangled_name);
  }
  return Status::OK();
}
```
### `LlvmCodeGen::LoadFunction()` 函数详细解释

这是一个 **Impala 后端（Backend）核心函数**，负责**为某个标量函数（scalar function）加载其真正的执行实现**，并返回一个 LLVM `llvm::Function*` 对象，以便后续用于代码生成（codegen）或直接调用。

它的作用是**统一处理所有类型的标量函数**（内置函数、Native UDF、IR UDF），并将它们转换为 LLVM 可以理解和优化的形式。

#### 函数签名解析

```cpp
Status LoadFunction(
    const TFunction& fn,                  // Thrift 定义的函数元数据（来自 FE）
    const string& symbol,                 // 函数符号名（可读名或 mangled 名）
    const ColumnType* return_type,        // 返回类型（可能为 NULL）
    const vector<ColumnType>& arg_types,  // 所有参数类型
    int num_fixed_args,                   // 固定参数数量
    bool has_varargs,                     // 是否有可变参数
    llvm::Function** llvm_fn,             // 输出：生成的 LLVM Function*
    LibCacheEntry** cache_entry)          // 输出：库缓存条目（用于 native 函数）
```

#### 主要逻辑分支（三大路径）

函数根据 `fn.binary_type` 分成 **三种处理路径**：

---

### 路径 1：Native UDF 或 部分“有问题的”Builtin（使用 native 实现）

```cpp
if (fn.binary_type == TFunctionBinaryType::NATIVE
    || (fn.binary_type == TFunctionBinaryType::BUILTIN && broken_builtin))
```

- **适用场景**：
  - 用户自定义的 Native UDF（编译成 `.so` 文件）。
  - 少数几个**无法 JIT 编译的内置函数**（`broken_builtin == true`），例如：
    - `from_utc_timestamp` / `to_utc_timestamp`：包含内联汇编（inline ASM），LLVM JIT 不支持。
    - `TimestampFunctions::AddSub`：包含 `try/catch`，JIT 环境下异常处理不工作。

- **处理方式**：
  1. 通过 `LibCache` 使用 `dlsym()` 从 `.so` 文件动态加载函数指针 `fn_ptr`。
  2. **手动构造一个 LLVM 函数原型（declaration）**，匹配 UDF 接口签名：
     - 返回类型：如果是 `DECIMAL`，按 x86-64 ABI 改为 `void`（通过隐藏的输出参数返回）。
     - 参数：
       - 第一个：`FunctionContext* ctx`
       - 后续：每个固定参数一个指针（如 `DoubleVal*`）
       - 可变参数：`int num_varargs` + `AnyVal* varargs`
  3. 使用 `execution_engine()->addGlobalMapping(*llvm_fn, fn_ptr)` 将这个 LLVM 函数**映射到真实的 native 函数地址**。
  4. 禁用 codegen 缓存（因为 native 函数地址可能在运行时变化，如 UDF 重载）。

- **结果**：`*llvm_fn` 是一个**函数声明**，没有函数体，但 LLVM 知道它的真实地址，调用时会跳转到 native 代码。

---

### 路径 2：普通 Builtin 函数（使用预编译 IR 实现）

```cpp
else if (fn.binary_type == TFunctionBinaryType::BUILTIN)
```

- **适用场景**：
  - 大多数内置函数（如 `abs()`、`lower()`、`now()`、`pi()` 等）。
  - 这些函数在构建时被交叉编译成 LLVM IR（位于 `impala_llvm_*.bc` 模块中）。

- **处理方式**：
  1. 调用 `GetFunction(symbol, true)`：
     - 在当前 LLVM module 中按 `symbol`（如 `"impala::MathFunctions::Abs"`）**直接查找函数**。
     - `true` 表示返回一个**克隆副本**（clone），因为后续可能需要修改（如内联常量）。
  2. 如果找不到 → 报错（说明版本不匹配）。
  3. 重命名函数为 demangled 可读名（便于调试）。

- **结果**：`*llvm_fn` 是从预编译 IR 模块中**克隆出的完整函数定义**（有函数体），可以被 LLVM 优化、内联、JIT 编译。

这是**现代 Impala 内置函数的主要执行路径**，也是“符号查找机制”的核心体现。

---

### 路径 3：IR UDF（用户提供的 LLVM IR 模块）

```cpp
else {  // fn.binary_type == TFunctionBinaryType::IR
```

- **适用场景**：
  - 用户上传的 IR UDF（`.ll` 或 `.bc` 文件，托管在 HDFS）。

- **处理方式**：
  1. 调用 `LinkModuleFromHdfs()` 将用户 IR 模块**链接（link）到当前查询的 LLVM module** 中。
  2. 然后同样调用 `GetFunction(symbol, true)` 查找并克隆函数。
  3. 重命名 demangled 名。

- **结果**：用户函数成为当前 module 的一部分，可被优化和 JIT。

---

### 关键技术点总结

| 技术点 | 说明 |
|--------|------|
| **符号查找** | 在路径2和3中，通过 `module_->getFunction(symbol)` 直接按**人类可读名**查找函数，这是摆脱 mangled 符号的关键。 |
| **懒加载（Lazy Materialization）** | `GetFunction()` 内部会调用 `MaterializeFunction()`，只在真正需要时解析 bitcode，节省内存。 |
| **函数克隆** | 使用 `CloneFunction()` 返回副本，允许安全修改（如常量内联），原始函数保持不变。 |
| **ABI 兼容** | 特殊处理 `DecimalVal` 返回值（x86-64 用隐藏指针参数）和 aarch64 的 `sret` 属性。 |
| **全局映射** | `addGlobalMapping()` 是 native 函数与 LLVM 世界连接的桥梁。 |

---

### 总结：这个函数做了什么？

`LoadFunction()` 是 Impala 标量函数执行的**统一入口**，它：

1. **识别函数类型**（Native / Builtin IR / IR UDF）。
2. **加载或查找函数实现**。
3. **构造或返回一个 LLVM `Function*`**，使其可以在 codegen 环境中被调用、优化、内联。
4. **屏蔽底层实现差异**，上层代码（如 `ScalarFnCall::GetCodegendComputeFnImpl`）只需关心如何包装调用即可。

正是这个函数实现了 Impala 从“硬编码 mangled 符号”到“动态符号查找 + IR 克隆”的演进，是整个 codegen 系统灵活性和高性能的关键基石。
*******************************************************************************************
enum TFunctionBinaryType {
  BUILTIN = 0    // 内置函数（IR 或 native）
  JAVA = 1       // Java UDF（已废弃）
  NATIVE = 2     // Native UDF（.so）
  IR = 3         // IR UDF（.ll/.bc）
}
****************************************************





