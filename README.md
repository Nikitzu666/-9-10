Выполнил: Смирнов Н.М.

Группа: ЭВТ-70

Игровой движок: Unity 2021.3.15

Название работы: разработка игры 2048

Лабораторная работа № 9-10
Тема: разработка игрового проекта 2048

Цель: приобрести навыки в разработке игрового проекта 2048

Ход работы

Выполнение работы

Импортирование ресурсов игры

![image](https://user-images.githubusercontent.com/119733911/205498791-48cf63e0-179c-459f-bfb6-fa0648292986.png)
Рисунок 9.1 - Папка Sprites

Размещение игровых объектов на сцене
![image](https://user-images.githubusercontent.com/119733911/205498802-fc3b3e93-a67e-4fc8-8b73-c5ad104189e6.png)
Рисунок 9.2 – Вид из окна Game

Когда игрок набрал сумму 32, то он выигрывает
![image](https://user-images.githubusercontent.com/119733911/205498809-e0cb41ce-57df-4b8e-b138-90f3f1307555.png)
Рисунок 9.3 – Окно выигрыша 


Разработка геймплея игры
Листинг 9.1. GameManager.cs
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using System.Security.Cryptography;
using DG.Tweening;
using UnityEngine;
using Random = UnityEngine.Random;

public class GameManager : MonoBehaviour {
    [SerializeField] private int _width = 4;
    [SerializeField] private int _height = 4;
    [SerializeField] private Node _nodePrefab;
    [SerializeField] private Block _blockPrefab;
    [SerializeField] private SpriteRenderer _boardPrefab;
    [SerializeField] private List<BlockType> _types;
    [SerializeField] private float _travelTime = 0.2f;
    [SerializeField] private int _winCondition = 2048;

    [SerializeField] private GameObject _winScreen, _loseScreen;

    private List<Node> _nodes;
    private List<Block> _blocks;
    private GameState _state;
    private int _round;

    private BlockType GetBlockTypeByValue(int value) => _types.First(t => t.Value == value);
    
    void Start() {
        ChangeState(GameState.GenerateLevel);
    }

    private void ChangeState(GameState newState) {
        _state = newState;

        switch (newState) {
            case GameState.GenerateLevel:
                GenerateGrid();
                break;
            case GameState.SpawningBlocks:
                SpawnBlocks(_round++ == 0 ? 2 : 1);
                break;
            case GameState.WaitingInput:
                break;
            case GameState.Moving:
                break;
            case GameState.Win:
                _winScreen.SetActive(true);
                break;
            case GameState.Lose:
                _loseScreen.SetActive(true);
                break;
            default:
                throw new ArgumentOutOfRangeException(nameof(newState), newState, null);
        }
    }

    void Update() {
        if(_state != GameState.WaitingInput) return;

        if(Input.GetKeyDown(KeyCode.LeftArrow)) Shift(Vector2.left);
        if(Input.GetKeyDown(KeyCode.RightArrow)) Shift(Vector2.right);
        if(Input.GetKeyDown(KeyCode.UpArrow)) Shift(Vector2.up);
        if(Input.GetKeyDown(KeyCode.DownArrow)) Shift(Vector2.down);
    }
    
    void GenerateGrid() {
        _round = 0;
        _nodes = new List<Node>();
        _blocks = new List<Block>();
        for (int x = 0; x < _width; x++) {
            for (int y = 0; y < _height; y++) {
                var node = Instantiate(_nodePrefab, new Vector2(x,y), Quaternion.identity);
                _nodes.Add(node);
            }
        }

        var center = new Vector2((float) _width /2 - 0.5f, (float) _height / 2 -0.5f);

        var board = Instantiate (_boardPrefab, center, Quaternion.identity);
        board.size = new Vector2(_width,_height);
    
        Camera.main.transform.position = new Vector3(center.x,center.y,-10);
    
        ChangeState(GameState.SpawningBlocks);
    }
    
    void SpawnBlocks(int amount) {
        
        var freeNodes = _nodes.Where(n=>n.OccupiedBlock == null).OrderBy(b => Random.value);
       
        foreach (var node in freeNodes.Take(amount)) {
            SpawnBlock(node, Random.value > 0.8f ? 4 : 2);
        }

        if (freeNodes.Count() == 1) {
            ChangeState(GameState.Lose);
            return;
        }

        ChangeState(_blocks.Any(b => b.Value == _winCondition) ? GameState.Win : GameState.WaitingInput);
    }    

    void SpawnBlock(Node node,int value) {
        var block = Instantiate(_blockPrefab, node.Pos, Quaternion.identity);
        block.Init(GetBlockTypeByValue(value));
        block.SetBlock(node);
        _blocks.Add(block);
    }

    void Shift(Vector2 dir) {
        ChangeState(GameState.Moving);
        
        var orderedBlocks = _blocks.OrderBy(b => b.Pos.x).ThenBy(b => b.Pos.y).ToList();
        if (dir == Vector2.right || dir == Vector2.up) orderedBlocks.Reverse();

        foreach (var block in orderedBlocks) {
            var next = block.Node;
            do {
                block.SetBlock(next);

                var possibleNode = GetNodeAtPosition(next.Pos + dir);
                if (possibleNode != null) {
                    // We know a node is present
                    // If it's possible to merge, set merge
                    if (possibleNode.OccupiedBlock != null && possibleNode.OccupiedBlock.CanMerge(block.Value)) {
                        block.MergeBlock(possibleNode.OccupiedBlock);
                    }
                    // Otherwise, cam we move to this spot?
                    else if (possibleNode.OccupiedBlock == null) next = possibleNode;

                    // None hit? End do while loop
                }


            } while (next != block.Node) ;

            

        }


        var sequence = DOTween.Sequence();

        foreach (var block in orderedBlocks) {
            var movePoint = block.MergingBlock != null ? block.MergingBlock.Node.Pos : block.Node.Pos;

            sequence.Insert(0, block.transform.DOMove(block.Node.Pos, _travelTime));
        }

        sequence.OnComplete(() => {
            foreach (var block in orderedBlocks.Where(b => b.MergingBlock != null)) {           
                MergeBlocks(block.MergingBlock,block);
            }

            ChangeState(GameState.SpawningBlocks);
        });



    }

    void MergeBlocks(Block baseBlock, Block mergingBlock) {
        SpawnBlock(baseBlock.Node, baseBlock.Value * 2);

        RemoveBlock(baseBlock);
        RemoveBlock(mergingBlock);
    }

    void RemoveBlock(Block block) {
        _blocks.Remove(block);
        Destroy(block.gameObject);
        
    }   
    Node GetNodeAtPosition(Vector2 pos) {
        return _nodes.FirstOrDefault(n => n.Pos == pos);
    }

}

[Serializable]

public struct BlockType {
    public int Value;
    public Color Color;
}

public enum GameState {
    GenerateLevel,
    SpawningBlocks,
    WaitingInput,
    Moving,
    Win,
    Lose
}


Листинг 9.2. Block.cs
using System.Collections;
using System.Collections.Generic;
using TMPro;
using UnityEngine;

public class Block : MonoBehaviour {
    public int Value;
    public Node Node;
    public Block MergingBlock;
    public bool Merging;
    public Vector2 Pos => transform.position;
    [SerializeField] private SpriteRenderer _renderer;
    [SerializeField] private TextMeshPro _text;
    public void Init(BlockType type) {
        Value = type.Value;
        _renderer.color = type.Color;
        _text.text = type.Value.ToString();
    }

    public void SetBlock(Node node) {
        if (Node != null) Node.OccupiedBlock = null;
        Node = node;
        Node.OccupiedBlock = this;
    }

    public void MergeBlock(Block blockToMergeWith) {
        // Set the block we are merning with
        MergingBlock = blockToMergeWith;

        // Set current node as unoccupied to allow blocks to use it
        Node.OccupiedBlock = null;

        // Set tbe base block as merging, so it does not get used twice.
        blockToMergeWith.Merging = true;
    }

    public bool CanMerge(int value) => value == Value && !Merging && MergingBlock == null;
}
Листинг 9.3. Node.cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Node : MonoBehaviour {
    
    public Vector2 Pos => transform.position;
    
    public Block OccupiedBlock;
} 


2. Вывод
В ходе лабораторной работы, были получены навыки в разработке игрового проекта 2048.
