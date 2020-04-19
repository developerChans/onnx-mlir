# How to support to process a new ONNX Operation

The text below describe the list of actions needed to successfully process a new ONNX operation. We are adding here support for an existing operation, thus when parsing the ONNX list of operation form the ONNX repository, this operation is already present. The step below explain how to add the necessary support to process that operation and lower it to code that can be executed using the LLVM compiler backend.

In the example below, we assume that we add support for the Concat ONNX operation. All paths are relatives to the root directory of onnx-mlir repository.


## Generate the proper ONNX.td.inc

The first step is to add support so that MLIR can determine the output type and shape from its input variables and parameters. This step is called “Shape Inference.” The first step is to indicate in the automatically generated `ONNXOps.td.inc` that the new operation needs to implement the shape inference interface method.

In the `utils/gen_doc.py` file, locate the `OpsWithShapeInference` python list and add your operation to it. For concat, we added the following:

```
OpsWithShapeInference = [
    'Exp', 'Tanh', 'Sinh', 'Cosh', 'Sigmoid', 'Relu', 'Add', 'Mul', 'Div',
    'Sub', 'And', 'Or', 'Xor', 'Sum', 'Max', 'Min', 'MatMul', 'Gemm',
    'LeakyRelu', 'Elu', 'Selu', 'HardSigmoid', 'Reshape', 'Reciprocal',
    'Identity', 'Cos', 'Log', 'Transpose', 'Softmax', 'ReduceMax', 'ReduceMin',
    'ReduceProd', 'ReduceSum', 'Softplus', 'Softsign', 'Sqrt', 'Unsqueeze',
    'Sign', 'Constant', 'AveragePool', 'Abs', 'Conv', 'Concat'
]
```

If the new operation needs support for other special handling, such as support for canonical forms or special parsing tools, the new operation will have to be added to such lists as well.

The next step will be to invoke the modified `gen_doc.py` file. For this operation, consult the help [here](https://github.com/onnx/onnx-mlir/blob/master/docs/ImportONNXDefs.md)

## Add shape inference

You will need to add code in the `src/Dialect/ONNX/ONNXOps.cpp.` Best is to look at other operations to get the general pattern. In general, the function return `true` in case of success, and `false` otherwise. User input errors (such as unsupported type, unknown option,…) should be flagged to the user using `emitError` using a friendly explanation of the error.

If your operation has a parameter ` dilations`, then uses the function ` dilations()` to provide the current value of that parameter. Parameter values can be set too; for example, in the `concat` operation, negative `axis` indicates an axis where counting goes from right to left. In this case, we can choose to normalize all values of the parameter `axis` to be from the usual left to right direction. Use `axisAttr(<<new attribute value>>)` to set the `axis` to its expected value. 

For references, all variables and parameters have getter functions, all parameters have setter functions as well.

## Invoke shape inference and test it. 

Next step is to enable onnx-mlir to invoke the new shape inference for this new operation. This is done by adding the operation name in function ` returnsDynamicShape` in file `Transform/ONNX/ShapeInferencePass.cpp.`

Once it is invoked, you will need to add literal tests in ` test/mlir/onnx/onnx_shape_inference.mlir.` Tests are executed by the `make check-onnx-lit` command in the build directory or by a `build/bin/onnx-mlir-opt` with parameter listed in the first line of the corresponding test mlir file.

## Lowering to krnl dialect

Files related to the lowering of the new operations resides in the `src/Conversion/ONNXtoKRNL` directory and subdirectories. For the `concat` operation, we added code to lower it to krnl dialect in the ` src/Conversion/ONNXToKrnl/Tensor/concat.cpp` file. See other similar lowering for inspiration. We suggest to use `assert` statements for any unexpected values encountered while lowering the operation, as illegal parameter values should be caught in the shape inference phase, not successive passes such as lowering to the krnl dialect.

In that file, the `populateLoweringONNXConcatOpPattern` operation (where `Concat` would be replaced with the actual new operation) will need to be defined in ` src/Conversion/ONNXToKrnl/ONNXToKrnlCommon.hpp` and invoked in the ` runOnModule` function in the ` src/Conversion/ONNXToKrnl/ConvertONNXToKrnl.cpp` file.

To compile properly, you will also need to add the new `.cpp` file in the ` src/Conversion/ONNXToKrnl/CMakeLists.txt` file.

## Testing using ONNX backend tests

Locate the new operation’s test in the ` third_party/onnx/onnx/backend/test/case/node` test directory. You will need to deduce from the code the names of the test files generated by the python script. You will then need to add these strings in the ` test/backend/test.py` file.

## Update your operation’s status

The operation’s status should be updated in the [Sharing work file](https://github.com/onnx/onnx-mlir/blob/master/SharingWork.md).

Tests are executed by the `make check-onnx-backend` command in the build directory.


