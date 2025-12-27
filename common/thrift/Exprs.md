// A flattened representation of a tree of Expr nodes, obtained by depth-first
// traversal.
struct TExpr {
  1: required list<TExprNode> nodes
}
所以nodes数组的第一个元素是root，随后的节点是深度优先遍历的各个子孙节点