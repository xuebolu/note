|           Flag           | Value |                         Description                          |
| :----------------------: | :---: | :----------------------------------------------------------: |
|      BIT_FINAL_ARC       |   1   |            arc对应的label是某个term的最后一个字符            |
|       BIT_LAST_ARC       |   2   | 表示此Arc是节点的最后一条边，此边后面的内容就是下一个节点了  |
|     BIT_TARGET_NEXT      |   4   | 表示此边应用了target_next优化，bytes中的下一个Node就是此边的目标节点 |
|      BIT_STOP_NODE       |   8   |                  arc的target是一个终止节点                   |
|    BIT_ARC_HAS_OUTPUT    |  16   |                        arc有output值                         |
| BIT_ARC_HAS_FINAL_OUTPUT |  32   |                       arc有finalOutput                       |

