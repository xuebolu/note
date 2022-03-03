## 前缀树简单实现



```java
public class Trie {
    /**
     * 根节点
     */
    private Node root;

    public Trie() {
        this.root = new Node(false);
    }
    public void add(String word) {
        Node node = root;
        int wordLength = word.length();
        for (int index = 0; index < wordLength; index++) {
            char c = word.charAt(index);
            Map<Character, Node> nextNodes = node.next;
            nextNodes.putIfAbsent(c, new Node(false));
            node = nextNodes.get(c);
        }
        node.isWord = true;
    }

    public boolean search(String word) {
        if (StringUtil.isBlank(word)) {
            return false;
        }
        Node node = root;
        for (char c : word.toCharArray()) {
            if (node.next.containsKey(c)) {
                node = node.next.get(c);
            } else {
                return false;
            }
        }
        return node.isWord;
    }

    public List<String> suggest(String word) {
        List<String> suggestList = new ArrayList<>();
        if (StringUtil.isBlank(word)) {
            return suggestList;
        }
        Node node = root;
        // 先找到word最后一个字符所对应的Node
        for (char c : word.toCharArray()) {
            if (node.next.containsKey(c)) {
                node = node.next.get(c);
            } else {
                return null;
            }
        }
        // 判断找到的Node是否能代表一个词
        if (node.isWord) {
            suggestList.add(word);
        }
        // 遍历Node的叶子节点，将叶子节点中能代表完整的词的词返回
        this.searchSubNodeWord(node, word, suggestList);
        return suggestList;
    }

    private void searchSubNodeWord(Node node, String prefix, List<String> suggestList) {
        for (Map.Entry<Character, Node> entry : node.next.entrySet()) {
            Character character = entry.getKey();
            Node subNode = entry.getValue();
            String newPreFix = prefix + character;
            if (subNode.isWord) {
                suggestList.add(newPreFix);
            }
            this.searchSubNodeWord(subNode, newPreFix, suggestList);
        }
    }

    private static class Node {
        /**
         * 下一个节点
         */
        private Map<Character, Node> next;
        /**
         * 是否为一个完整的词
         */
        private boolean isWord;

        Node(boolean isWord) {
            this.next = new HashMap<>();
            this.isWord = isWord;
        }
    }
}
```

