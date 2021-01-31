# The Transformer: fairseq edition
*by Javier Ferrando*

The Transformer was presented in ["Attention is All You Need"](https://arxiv.org/abs/1706.03762) and introduced a new architecture for many NLP tasks. In this post we exhibit an explanation of the Transformer architecture on Neural Machine Translation focusing on the [fairseq](https://github.com/pytorch/fairseq) implementation. We believe this could be useful for researchers and developers starting out on this framework.

The blog is inspired by [The annotated Transformer](https://nlp.seas.harvard.edu/2018/04/03/attention.html), [The Illustrated Transformer](http://jalammar.github.io/illustrated-transformer/) and [Fairseq Transformer, BART](https://yinghaowang.xyz/technology/2020-03-14-FairseqTransformer.html) blogs.

## Model Architecture

The Transformer is based on a stack of encoders and another stack of decoders. The encoder maps an input sequence of tokens <!-- $$ \mathcal{X}=(token_{0},...,token_{src\_len}) $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Cmathcal%7BX%7D%3D(token_%7B0%7D%2C...%2Ctoken_%7Bsrc%5C_len%7D)"> to a sequence of continuous vector representations <!-- $$ encoder\_out = (encoder\_out_0,..., encoder\_out_{src\_len}) $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=encoder%5C_out%20%3D%20(encoder%5C_out_0%2C...%2C%20encoder%5C_out_%7Bsrc%5C_len%7D)">. Given <!-- $$ encoder\_out $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=encoder%5C_out">, the decoder then generates an output sequence <!-- $$ \mathcal{Y} = (output_0,...,output_{T}) $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Cmathcal%7BY%7D%20%3D%20(output_0%2C...%2Coutput_%7BT%7D)"> of symbols one element at a time. At each step the model is auto-regressive, consuming the previously generated symbols as additional input when generating the next token.

<p align="center">
    <img src="../assets/1_TheTransformer/transformer.png?raw=true" width="50%" align="center"/>
</p>

To see the general structure of the code in fairseq implementation I recommend reading [Fairseq Transformer, BART](https://yinghaowang.xyz/technology/2020-03-14-FairseqTransformer.html).

This model is implemented in fairseq as <code class="language-plaintext highlighter-rouge">TransformerModel</code> in [fairseq/models/transformer.py](https://github.com/pytorch/fairseq/blob/master/fairseq/models/transformer.py).


```python
class TransformerModel(FairseqEncoderDecoderModel):
...
  def forward(
          self,
          src_tokens,
          src_lengths,
          prev_output_tokens,
          return_all_hiddens: bool = True,
          features_only: bool = False,
          alignment_layer: Optional[int] = None,
          alignment_heads: Optional[int] = None,
      ):
          encoder_out = self.encoder(
              src_tokens, src_lengths=src_lengths, return_all_hiddens=return_all_hiddens
          )
          decoder_out = self.decoder(
              prev_output_tokens,
              encoder_out=encoder_out,
              features_only=features_only,
              alignment_layer=alignment_layer,
              alignment_heads=alignment_heads,
              src_lengths=src_lengths,
              return_all_hiddens=return_all_hiddens,
          )
          return decoder_out
```

## Encoder

The encoder (<code class="language-plaintext highlighter-rouge">TransformerEncoder</code>) is composed of a stack of <!-- $$ N=encoder\_layers $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=N%3Dencoder%5C_layers"> identical layers.

The encoder recieves a list of tokens <!-- $$ \mathcal{X}= $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Cmathcal%7BX%7D%3D"><code class="language-plaintext highlighter-rouge">src_tokens</code><!-- $$ =(token_{0},...,token_{src\_len}) $$ --><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%3D(token_%7B0%7D%2C...%2Ctoken_%7Bsrc%5C_len%7D)">  which are then converted to continuous vector representions <code class="language-plaintext highlighter-rouge">x = self.forward_embedding(src_tokens, token_embeddings)</code>, which is made of the sum of the (scaled) embedding lookup and the positional embedding: <code class="language-plaintext highlighter-rouge">x = self.embed_scale * self.embed_tokens(src_tokens) + self.embed_positions(src_tokens)</code>.

From now on, let's consider <!-- $$ X^L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=X%5EL"> as the <!-- $$ L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=L"> encoder layer input sequence. <!-- $$ X^{1} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=X%5E%7B1%7D"> refers then to the vectors representation of the input sequence tokens of the first layer, after computing <code class="language-plaintext highlighter-rouge">self.forward_embedding</code> on <code class="language-plaintext highlighter-rouge">src_tokens</code>.

<p align="center">
<img src="../assets/1_TheTransformer/operations.png?raw=true" width="45%" align="center"/>
</p>

Note that although <!-- $$ X^L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=X%5EL"> is represented in fairseq as a tensor of shape <code class="language-plaintext highlighter-rouge">src_len x batch x encoder_embed_dim</code>, for the sake of simplicity, we take <code class="language-plaintext highlighter-rouge">batch=1</code> in the upcoming mathematical notation and just consider it as a <code class="language-plaintext highlighter-rouge">src_len x encoder_embed_dim</code> matrix.


<!-- $$
X^L = \begin{bmatrix} x_{0}\\ \vdots\\ x_{src\_len} \end{bmatrix}
$$ --> 
<p align="center"><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=X%5EL%20%3D%20%5Cbegin%7Bbmatrix%7D%20x_%7B0%7D%5C%5C%20%5Cvdots%5C%5C%20x_%7Bsrc%5C_len%7D%20%5Cend%7Bbmatrix%7D"></p>

Where <!-- $$ x_{i} \in \mathbb{R}^{encoder\_embed\_dim} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=x_%7Bi%7D%20%5Cin%20%5Cmathbb%7BR%7D%5E%7Bencoder%5C_embed%5C_dim%7D">.


```python
class TransformerEncoder(FairseqEncoder):
...
  def forward(
        self,
        src_tokens,
        src_lengths,
        return_all_hiddens: bool = False,
        token_embeddings: Optional[torch.Tensor] = None,
    ):

        x, encoder_embedding = self.forward_embedding(src_tokens, token_embeddings)

        # batch x src_lengths x encoder_embed_dim
        #                     -> src_lengths x batch x encoder_embed_dim
        x = x.transpose(0, 1)

        # compute padding mask
        encoder_padding_mask = src_tokens.eq(self.padding_idx)

        encoder_states = [] if return_all_hiddens else None

        # encoder layers
        for layer in self.layers:
            x = layer(x, encoder_padding_mask)
            if return_all_hiddens:
                assert encoder_states is not None
                encoder_states.append(x)

        if self.layer_norm is not None:
            x = self.layer_norm(x)

        return EncoderOut(
            encoder_out=x,  # src_lengths x batch x encoder_embed_dim
            encoder_padding_mask=encoder_padding_mask,
            encoder_embedding=encoder_embedding,
            encoder_states=encoder_states, # List[src_lengths x batch x encoder_embed_dim]
            src_tokens=None,
            src_lengths=None,
        )
```

This returns a NamedTuple object <code class="language-plaintext highlighter-rouge">encoder_out</code>.

* encoder_out: of shape <code class="language-plaintext highlighter-rouge">src_len x batch x encoder_embed_dim</code>, the last layer encoder's embedding which, as we will see, is used by the Decoder. Note that is the same as <!-- $$ X^{N+1} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=X%5E%7BN%2B1%7D"> when <code class="language-plaintext highlighter-rouge">batch=1</code>.
* encoder_padding_mask: of shape <code class="language-plaintext highlighter-rouge">batch x src_len</code>. Binary ByteTensor where padding elements are indicated by 1.
* encoder_embedding: of shape <code class="language-plaintext highlighter-rouge">src_len x batch x encoder_embed_dim</code>, the words (scaled) embedding lookup.
* encoder_states: of shape <code class="language-plaintext highlighter-rouge">list[src_len x batch x encoder_embed_dim]</code>, intermediate enocoder layer's output.


### Encoder Layer

The previous snippet of code shows a loop over the layers of the Encoder block, <code class="language-plaintext highlighter-rouge">for layer in self.layers</code>. This layer is implemented in fairseq in <code class="language-plaintext highlighter-rouge">class TransformerEncoderLayer(nn.Module)</code> inside [fairseq/modules/transformer_layer.py](https://github.com/pytorch/fairseq/blob/master/fairseq/modules/transformer_layer.py) and computes the following operations:

<p align="center">
<img src="../assets/1_TheTransformer/encoder.png?raw=true" width="25%" align="center"/>
</p>

The input of the encoder layer is passed through the self-attention module <code class="language-plaintext highlighter-rouge">self.self_attn</code>, dropout (<code class="language-plaintext highlighter-rouge">self.dropout_module(x)</code>) is then applied before getting to the Residual & Normalization module (made of a residual connection <code class="language-plaintext highlighter-rouge">self.residual_connection(x, residual)</code> and a layer normalization (LayerNorm) <code class="language-plaintext highlighter-rouge">self.self_attn_layer_norm(x)</code>


```python
class TransformerEncoderLayer(nn.Module):
...
  def forward(self, x, encoder_padding_mask, attn_mask: Optional[Tensor] = None):
    if attn_mask is not None:
      attn_mask = attn_mask.masked_fill(attn_mask.to(torch.bool), -1e8)

    residual = x
    if self.normalize_before:
        x = self.self_attn_layer_norm(x)
    x, _ = self.self_attn(
        query=x,
        key=x,
        value=x,
        key_padding_mask=encoder_padding_mask,
        attn_mask=attn_mask,
    )
    x = self.dropout_module(x)
    x = self.residual_connection(x, residual)
    if not self.normalize_before:
        x = self.self_attn_layer_norm(x)
```

Then, the result is passed through a position-wise feed-forward network composed by two fully connected layers, <code class="language-plaintext highlighter-rouge">fc1</code> and <code class="language-plaintext highlighter-rouge">fc2</code> with a ReLU activation in between (<code class="language-plaintext highlighter-rouge">self.activation_fn(self.fc1(x))</code>) and dropout <code class="language-plaintext highlighter-rouge">self.dropout_module(x)</code>.

<!-- $$
\text{Feed Forward}(x)=\max(0, xW_1 + b_1) W_2 + b_2
$$ -->
<p align="center"><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Ctext%7BFeed%20Forward%7D(x)%3D%5Cmax(0%2C%20xW_1%20%2B%20b_1)%20W_2%20%2B%20b_2" class="center"></p>

```python
    residual = x
    if self.normalize_before:
        x = self.final_layer_norm(x)

    x = self.activation_fn(self.fc1(x))
    x = self.activation_dropout_module(x)
    x = self.fc2(x)
    x = self.dropout_module(x)
       
```

Finally, a residual connection is made before another layer normalization layer.


```python
    x = self.residual_connection(x, residual)
    if not self.normalize_before:
        x = self.final_layer_norm(x)
    return x
```

### Self-attention

As we have seen, the input of each encoder layer is firstly passed through a self-attention layer ([fairseq/modules/multihead_attention.py](https://github.com/pytorch/fairseq/blob/master/fairseq/modules/multihead_attention.py))


```python
class MultiheadAttention(nn.Module):
...
  def forward(
        self,
        query,
        key: Optional[Tensor],
        value: Optional[Tensor],
        key_padding_mask: Optional[Tensor] = None,
        incremental_state: Optional[Dict[str, Dict[str, Optional[Tensor]]]] = None,
        need_weights: bool = True,
        static_kv: bool = False,
        attn_mask: Optional[Tensor] = None,
        before_softmax: bool = False,
        need_head_weights: bool = False,
    ) -> Tuple[Tensor, Optional[Tensor]]:
```

Each encoder layer input <!-- $$ X^L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=X%5EL">, shown as <code class="language-plaintext highlighter-rouge">query</code> below since three identical copies are passed to the self-attention module, is multiplied by three weight matrices learned during the training process: <!-- $$ W^{Q}, W^{K} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5E%7BQ%7D%2C%20W%5E%7BK%7D"> and <!-- $$ W^{V} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5E%7BV%7D">, obtaining <!-- $$ Q, K $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=Q%2C%20K"> and <!-- $$ V $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=V">. Each row of this output matrices represents the query, key and value vectors of each token in the sequence, represented as <!-- $$ q, k $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=q%2C%20k"> and <!-- $$ v $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=v"> in the formulas that follow.


```python
    if self.self_attention:
      q = self.q_proj(query) # Q
      k = self.k_proj(query) # K
      v = self.v_proj(query) # V
    q *= self.scaling
```

The self-attention module does the following operation:

<!-- $$
\mathrm{softmax}(\frac{QK^\top}{\sqrt{d_{k}}})V
$$ --> 
<p align="center"><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Cmathrm%7Bsoftmax%7D(%5Cfrac%7BQK%5E%5Ctop%7D%7B%5Csqrt%7Bd_%7Bk%7D%7D%7D)V" class="center"></p>

```python
    attn_weights = torch.bmm(q, k.transpose(1, 2)) # QK^T multiplication
```

Given a token in the input, <!-- $$ i \in X^L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=i%20%5Cin%20X%5EL">, it is passed to the self-attention function. Then, by means of dot products, scalar values (scores) are obtained between the query vector <!-- $$ q_{i} = iW^Q $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=q_%7Bi%7D%20%3D%20iW%5EQ"> and every key vector of the input sequence <!-- $$ k_{j} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=k_%7Bj%7D">. The intuition is that this performs a similarity operation, similar queries and keys vectors will yield higher scores.

These scores represent how much attention is paid by the self-attention layer to other parts of the sequence when encoding <!-- $$ i $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=i">. By multiplying <!-- $$ q_{i} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=q_%7Bi%7D"> by the matrix <!-- $$ K^{T} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=K%5E%7BT%7D">, a list of <code class="language-plaintext highlighter-rouge">src_len</code> scores is output. The scores are then passed through a softmax function giving bounded values:


<!-- $$
\alpha_{i} = \text{softmax}(\frac{\mathbf{q}_i {K}^\top}{\sqrt{d_k}}) = \frac{\exp(\frac{\mathbf{q}_i {K}^\top}{\sqrt{d_k}})}{ \sum_{j=0}^{src\_len} \exp(\frac{\mathbf{q}_i k_{j}^\top}{\sqrt{d_k}})}
$$ --> 
<p align="center"><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Calpha_%7Bi%7D%20%3D%20%5Ctext%7Bsoftmax%7D(%5Cfrac%7B%5Cmathbf%7Bq%7D_i%20%7BK%7D%5E%5Ctop%7D%7B%5Csqrt%7Bd_k%7D%7D)%20%3D%20%5Cfrac%7B%5Cexp(%5Cfrac%7B%5Cmathbf%7Bq%7D_i%20%7BK%7D%5E%5Ctop%7D%7B%5Csqrt%7Bd_k%7D%7D)%7D%7B%20%5Csum_%7Bj%3D0%7D%5E%7Bsrc%5C_len%7D%20%5Cexp(%5Cfrac%7B%5Cmathbf%7Bq%7D_i%20k_%7Bj%7D%5E%5Ctop%7D%7B%5Csqrt%7Bd_k%7D%7D)%7D" class="center"></p>



```python
    attn_weights_float = utils.softmax(
                attn_weights, dim=-1, onnx_trace=self.onnx_trace
            )
    attn_weights = attn_weights_float.type_as(attn_weights)
```

The division by the square root of the dimension of the key vectors <!-- $$ d_{k} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=d_%7Bk%7D"> (for getting more stable gradients) is done previously <code class="language-plaintext highlighter-rouge">q *= self.scaling</code> instead in fairseq.


For example, given the sentence "the nice cat walks away from us" for the token <!-- $$ i=\text{from} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=i%3D%5Ctext%7Bfrom%7D">, its corresponding attention weights <!-- $$ \alpha_{i} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Calpha_%7Bi%7D"> for every other token <!-- $$ j $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=j"> in the input sequence could be:

<p align="center">
<img src="../assets/1_TheTransformer/probs.jpg?raw=true" width="50%" align="center"/>
</p>

Once we have normalized scores for every pair of tokens <!-- $$ \{i,j\} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5C%7Bi%2Cj%5C%7D">, we multiply these weights by the value vector <!-- $$ v_{j} \forall j \in X^L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=v_%7Bj%7D%20%5Cforall%20j%20%5Cin%20X%5EL"> (each row in matrix <!-- $$ V $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=V">) and finally sum up those vectors:

<!-- $$
z_{i} = \sum_{j=0}^{src\_len}\alpha_{i,j}v_{j}
$$ --> 
<p align="center"><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=z_%7Bi%7D%20%3D%20%5Csum_%7Bj%3D0%7D%5E%7Bsrc%5C_len%7D%5Calpha_%7Bi%2Cj%7Dv_%7Bj%7D" class="center"></p>

Where <!-- $$ z_{i} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=z_%7Bi%7D"> represents row <!-- $$ i $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=i"> of <!-- $$ Z $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=Z">. By doing the matrix multiplication of the attention weight matrix <code class="language-plaintext highlighter-rouge">attn_weights</code> and <!-- $$ V $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=V">, <!-- $$ \mathrm{softmax}(\frac{QK^{T}}{\sqrt{d_k}})V $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Cmathrm%7Bsoftmax%7D(%5Cfrac%7BQK%5E%7BT%7D%7D%7B%5Csqrt%7Bd_k%7D%7D)V">, we directly get matrix <!-- $$ Z $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=Z">.


```python
    attn_probs = self.dropout_module(attn_weights)
    assert v is not None
    attn = torch.bmm(attn_probs, v)
```

This process is done in parallel in each of the self-attention heads. So, in total <code class="language-plaintext highlighter-rouge">encoder_attention_heads</code> matrices are output. Each head has its own <!-- $$ W^{Q}, W^{K} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5E%7BQ%7D%2C%20W%5E%7BK%7D"> and <!-- $$ W^{V} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5E%7BV%7D"> weight matrices which are randomly initialized, so the result leads to different representation subspaces in each of the self-attention heads.

The output matrices <!-- $$ Z $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=Z"> of every self-attention head are concatenated into a single one to which a linear transformation <!-- $$ W^{O} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5E%7BO%7D"> (<code class="language-plaintext highlighter-rouge">self.out_proj</code>) is applied:

<!-- $$
attn = Concat(Z^{head_{i}},\cdots,Z^{head_{h}})W^{O}
$$ --> 
<p align="center"><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=attn%20%3D%20Concat(Z%5E%7Bhead_%7Bi%7D%7D%2C%5Ccdots%2CZ%5E%7Bhead_%7Bh%7D%7D)W%5E%7BO%7D" class="center"></p>

```python
    #concatenating each head representation before W^o projection
    attn = attn.transpose(0, 1).contiguous().view(tgt_len, bsz, embed_dim)
    #W^o projection
    attn = self.out_proj(attn)
    attn_weights: Optional[Tensor] = None
    if need_weights:
        attn_weights = attn_weights_float.view(
            bsz, self.num_heads, tgt_len, src_len
        ).transpose(1, 0)
        if not need_head_weights:
            # average attention weights over heads
            attn_weights = attn_weights.mean(dim=0)

    return attn, attn_weights
```

Notice that <code class="language-plaintext highlighter-rouge">attn_probs</code> has dimensions (bsz * self.num_heads, tgt_len, src_len)


To facilitate these residual connections, all sub-layers in the model, as well as the embedding layers, produce outputs of dimension <code class="language-plaintext highlighter-rouge">encoder_embed_dim</code>.

## Decoder

The decoder is composed of a stack of <!-- $$ N=decoder\_layers $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=N%3Ddecoder%5C_layers"> identical layers.

The goal of the decoder is to generate a sequence <!-- $$ \mathcal{Y} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Cmathcal%7BY%7D"> in the target language. The <code class="language-plaintext highlighter-rouge">TransformerDecoder</code> inherits from <code class="language-plaintext highlighter-rouge">FairseqIncrementalDecoder</code>. It differs from the encoder in that it performs incremental decoding. This means that at each time step <!-- $$ t $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=t"> a forward pass is done through the decoder, generating <!-- $$ output_{t} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=output_%7Bt%7D">, which is then fed as input to the next time step decoding process.

The encoder output <code class="language-plaintext highlighter-rouge">encoder_out.encoder_out</code> is used by the decoder (in each layer) together with <!-- $$ \mathcal{Y}<t=(output_{0},...,output_{t-1}) $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Cmathcal%7BY%7D%3Ct%3D(output_%7B0%7D%2C...%2Coutput_%7Bt-1%7D)"> (<code class="language-plaintext highlighter-rouge">prev_output_tokens</code>) to generate one feature vector per target token at each time step (<code class="language-plaintext highlighter-rouge">tgt_len = 1</code> in each forward pass). This feature vector is then transformed by a linear layer and passed through a softmax layer <code class="language-plaintext highlighter-rouge">self.output_layer(x)</code> to get a probability distribution over the target language vocabulary.

Following the beam search algorithm, top <code class="language-plaintext highlighter-rouge">beam</code> hypotheses are chosen and inserted in the batch dimension input of the decoder (<code class="language-plaintext highlighter-rouge">prev_output_tokens</code>) for the next time step.

<p align="center">
<img src="../assets/1_TheTransformer/decoder.png?raw=true" width="35%" align="center"/>
</p>

We consider <!-- $$ query^L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=query%5EL"> as the <!-- $$ L $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=L"> decoder layer input sequence. <!-- $$ query^{1} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=query%5E%7B1%7D"> refers then to the vector representation of the input sequence tokens of the first layer, after computing <code class="language-plaintext highlighter-rouge">self.forward_embedding</code> on <code class="language-plaintext highlighter-rouge">prev_output_tokens</code>. Note that here <code class="language-plaintext highlighter-rouge">self.forward_embedding</code> is not defined, but we refer to <code class="language-plaintext highlighter-rouge">self.embed_tokens(prev_output_tokens)</code> and <code class="language-plaintext highlighter-rouge">self.embed_positions(prev_output_tokens)</code>.


```python
class TransformerDecoder(FairseqIncrementalDecoder):
...
  def forward(
        self,
        prev_output_tokens,
        encoder_out: Optional[EncoderOut] = None,
        incremental_state: Optional[Dict[str, Dict[str, Optional[Tensor]]]] = None,
        features_only: bool = False,
        full_context_alignment: bool = False,
        alignment_layer: Optional[int] = None,
        alignment_heads: Optional[int] = None,
        src_lengths: Optional[Any] = None,
        return_all_hiddens: bool = False,
    ):
        
    x, extra = self.extract_features(
        prev_output_tokens,
        encoder_out=encoder_out,
        incremental_state=incremental_state,
        full_context_alignment=full_context_alignment,
        alignment_layer=alignment_layer,
        alignment_heads=alignment_heads,
    )
    if not features_only:
        x = self.output_layer(x)
    return x, extra
```


```python
def extract_features(
        self,
        prev_output_tokens,
        encoder_out: Optional[EncoderOut] = None,
        incremental_state: Optional[Dict[str, Dict[str, Optional[Tensor]]]] = None,
        full_context_alignment: bool = False,
        alignment_layer: Optional[int] = None,
        alignment_heads: Optional[int] = None,
    ):
    return self.extract_features_scriptable(
        prev_output_tokens,
        encoder_out,
        incremental_state,
        full_context_alignment,
        alignment_layer,
        alignment_heads,
    )
```

In the first time step, <code class="language-plaintext highlighter-rouge">prev_output_tokens</code> represents the beginning of sentence (BOS) token index. Its embedding enters the decoder as a tensor <code class="language-plaintext highlighter-rouge">beam*batch x tgt_len x encoder_embed_dim</code>.


```python
def extract_features_scriptable(
        self,
        prev_output_tokens,
        encoder_out: Optional[EncoderOut] = None,
        incremental_state: Optional[Dict[str, Dict[str, Optional[Tensor]]]] = None,
        full_context_alignment: bool = False,
        alignment_layer: Optional[int] = None,
        alignment_heads: Optional[int] = None,
    ):
  ..
    positions = (
            self.embed_positions(
                prev_output_tokens, incremental_state=incremental_state
            )
    x = self.embed_scale * self.embed_tokens(prev_output_tokens)
    if positions is not None:
            x += positions
    attn: Optional[Tensor] = None
    inner_states: List[Optional[Tensor]] = [x]
    for idx, layer in enumerate(self.layers):
        if incremental_state is None and not full_context_alignment:
            self_attn_mask = self.buffered_future_mask(x)
        else:
            self_attn_mask = None

        x, layer_attn, _ = layer(
            x,
            encoder_out.encoder_out if encoder_out is not None else None,
            encoder_out.encoder_padding_mask if encoder_out is not None else None,
            incremental_state,
            self_attn_mask=self_attn_mask,
            self_attn_padding_mask=self_attn_padding_mask,
            need_attn=bool((idx == alignment_layer)),
            need_head_weights=bool((idx == alignment_layer)),
        )
        inner_states.append(x)
        if layer_attn is not None and idx == alignment_layer:
            attn = layer_attn.float().to(x)

    if attn is not None:
        if alignment_heads is not None:
            attn = attn[:alignment_heads]

        # average probabilities over heads
        attn = attn.mean(dim=0)

    if self.layer_norm is not None:
        x = self.layer_norm(x)

    # T x B x C -> B x T x C
    x = x.transpose(0, 1)

    if self.project_out_dim is not None:
        x = self.project_out_dim(x)

    return x, {"attn": [attn], "inner_states": inner_states}
```

### Decoder layer

The previous snippet of code shows a loop over the layers of the Decoder block <code class="language-plaintext highlighter-rouge">for idx, layer in enumerate(self.layers):</code>. This layer is implemented in fairseq in <code class="language-plaintext highlighter-rouge">class TransformerDecoderLayer(nn.Module)</code> inside [fairseq/modules/transformer_layer.py](https://github.com/pytorch/fairseq/blob/master/fairseq/modules/transformer_layer.py) and computes the following operations:

<p align="center">
<img src="../assets/1_TheTransformer/incremental_decoding.png?raw=true" width="40%" align="center"/>
</p>

In addition to the two sub-layers in each encoder layer, the decoder inserts a third sub-layer (Encoder-Decoder Attention), which performs multi-head attention over the output of the encoder stack as input for <!-- $$ W^{K} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5E%7BK%7D"> and <!-- $$ W^{V} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5E%7BV%7D"> and the ouput of the previous module <!-- $$ attn_{t}* $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=attn_%7Bt%7D*">.  Similar to the encoder, it employs residual connections around each of the sub-layers, followed by layer normalization.


```python
class TransformerDecoderLayer(nn.Module):
    ..
    def forward(
        self,
        x,
        encoder_out: Optional[torch.Tensor] = None,
        encoder_padding_mask: Optional[torch.Tensor] = None,
        incremental_state: Optional[Dict[str, Dict[str, Optional[Tensor]]]] = None,
        prev_self_attn_state: Optional[List[torch.Tensor]] = None,
        prev_attn_state: Optional[List[torch.Tensor]] = None,
        self_attn_mask: Optional[torch.Tensor] = None,
        self_attn_padding_mask: Optional[torch.Tensor] = None,
        need_attn: bool = False,
        need_head_weights: bool = False,
    ):

        ...
        
        residual = x
        if self.normalize_before:
            x = self.self_attn_layer_norm(x)      
        
```

### Self-attention in the decoder


```python
...

        y = x

        x, attn = self.self_attn(
            query=x,
            key=y,
            value=y,
            key_padding_mask=self_attn_padding_mask,
            incremental_state=incremental_state, # previous keys and values stored here
            need_weights=False,
            attn_mask=self_attn_mask,
        )
        x = self.dropout_module(x)
        x = self.residual_connection(x, residual)
        if not self.normalize_before:
            x = self.self_attn_layer_norm(x)
```

During incremental decoding, <!-- $$ (output_{0},...,output_{t-2}) $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=(output_%7B0%7D%2C...%2Coutput_%7Bt-2%7D)"> enter the self-attention module as <code class="language-plaintext highlighter-rouge">prev_key</code> and <code class="language-plaintext highlighter-rouge">prev_value</code> vectors that are stored in <code class="language-plaintext highlighter-rouge">incremental_state</code>. Since there is no need to recompute <!-- $$ K $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=K"> and <!-- $$ V $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=V"> every time, incremental decoding caches these values and concatenates with keys an values from <!-- $$ output_{t-1} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=output_%7Bt-1%7D">. Then, updated <!-- $$ K $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=K"> and <!-- $$ V $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=V"> are stored in <code class="language-plaintext highlighter-rouge">prev_key</code> and passed again to <code class="language-plaintext highlighter-rouge">incremental_state</code>.

<p align="center">
<img src="../assets/1_TheTransformer/decoder_self_attn.png?raw=true" width="65%" align="center"/>
</p>

The last time step output token in each decoding step, <!-- $$ output_{t-1} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=output_%7Bt-1%7D">, enters as a query after been embedded. So, queries here have one element in the second dimension, that is, there is no need to use matrix <!-- $$ Q $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=Q"> notation.
As before, scalar values (scores) <!-- $$ \alpha $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Calpha"> are obtained between the query vector <!-- $$ q_{t-1} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=q_%7Bt-1%7D"> and every key vector of the whole previous tokens sequence.

Flashing back to ([fairseq/modules/multihead_attention.py](https://github.com/pytorch/fairseq/blob/master/fairseq/modules/multihead_attention.py)) we can see how key and values are obtained inside the Multihead attention module and how these udates in  <code class="language-plaintext highlighter-rouge">saved_state</code> and  <code class="language-plaintext highlighter-rouge">incremental_state</code> are done:


```python
 class MultiheadAttention(nn.Module):
...
  def forward(
        ...
    ):
        ...
        if incremental_state is not None:
            saved_state = self._get_input_buffer(incremental_state) # getting saved_state
        ...
        if saved_state is not None:
            if "prev_key" in saved_state:
                _prev_key = saved_state["prev_key"]
                assert _prev_key is not None
                prev_key = _prev_key.view(bsz * self.num_heads, -1, self.head_dim)
                if static_kv: # in encoder-endoder attention
                    k = prev_key
                else:
                    assert k is not None
                    k = torch.cat([prev_key, k], dim=1) # concatenation of K
            if "prev_value" in saved_state:
                _prev_value = saved_state["prev_value"]
                assert _prev_value is not None
                prev_value = _prev_value.view(bsz * self.num_heads, -1, self.head_dim)
                if static_kv: # in encoder-endoder attention
                    v = prev_value
                else:
                    assert v is not None
                    v = torch.cat([prev_value, v], dim=1) # concatenation of V
            prev_key_padding_mask: Optional[Tensor] = None
            if "prev_key_padding_mask" in saved_state:
                prev_key_padding_mask = saved_state["prev_key_padding_mask"]
            assert k is not None and v is not None
            key_padding_mask = MultiheadAttention._append_prev_key_padding_mask(
                key_padding_mask=key_padding_mask,
                prev_key_padding_mask=prev_key_padding_mask,
                batch_size=bsz,
                src_len=k.size(1),
                static_kv=static_kv,
            )

            saved_state["prev_key"] = k.view(bsz, self.num_heads, -1, self.head_dim)
            saved_state["prev_value"] = v.view(bsz, self.num_heads, -1, self.head_dim)
            saved_state["prev_key_padding_mask"] = key_padding_mask
            # In this branch incremental_state is never None
            assert incremental_state is not None
            incremental_state = self._set_input_buffer(incremental_state, saved_state) # update
```

### Encoder-Decoder attention

The Encoder-Decoder attention receives key and values from the encoder output <code class="language-plaintext highlighter-rouge">encoder_out.encoder_out</code> and the query from the previous module <!-- $$ attn_{t}* $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=attn_%7Bt%7D*">. Here, <!-- $$ q_{t} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=q_%7Bt%7D"> is compared against every key vector received from the encoder (and transformed by <!-- $$ W^K $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=W%5EK">).

As before, <!-- $$ K $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=K"> and <!-- $$ V $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=V"> don't need to be recomputed every time step since they are constant for the whole decoding process. Encoder-Decoder attention uses <code class="language-plaintext highlighter-rouge">static_kv=True</code> so that there is no need to update the <code class="language-plaintext highlighter-rouge">incremental_state</code> (see previous code snippet).

Now, just one vector <!-- $$ z_{t} $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=z_%7Bt%7D"> is generated at each time step by each head as a weighted average of the <!-- $$ v $$ --> <img style="background: white;" src="https://render.githubusercontent.com/render/math?math=v"> vectors.

<p align="center"><img src="../assets/1_TheTransformer/decoder_enc_dec_attn.png?raw=true" width="45%" align="center"/>
</p>


```python
        if self.encoder_attn is not None and encoder_out is not None:
            residual = x
            if self.normalize_before:
                x = self.encoder_attn_layer_norm(x)
            ...
            
            x, attn = self.encoder_attn(
                query=x,
                key=encoder_out,
                value=encoder_out,
                key_padding_mask=encoder_padding_mask,
                incremental_state=incremental_state,
                static_kv=True,
                need_weights=need_attn or (not self.training and self.need_attn),
                need_head_weights=need_head_weights,
            )
            x = self.dropout_module(x)
            x = self.residual_connection(x, residual)
            if not self.normalize_before:
                x = self.encoder_attn_layer_norm(x)
```

As in the case of the encoder, the result is passed through a position-wise feed-forward network composed by two fully connected layers:

<!-- $$
\text{Feed Forward}(x)=\max(0, xW_1 + b_1) W_2 + b_2
$$ --> 
<p align="center"><img style="background: white;" src="https://render.githubusercontent.com/render/math?math=%5Ctext%7BFeed%20Forward%7D(x)%3D%5Cmax(0%2C%20xW_1%20%2B%20b_1)%20W_2%20%2B%20b_2" class="center"></p>

```python
        residual = x
        if self.normalize_before:
            x = self.final_layer_norm(x)

        x = self.activation_fn(self.fc1(x))
        x = self.activation_dropout_module(x)
        x = self.fc2(x)
        x = self.dropout_module(x)
        
```

Finally, a residual connection is made before another layer normalization layer.


```python
        x = self.residual_connection(x, residual)
        if not self.normalize_before:
            x = self.final_layer_norm(x)
...
        return x, attn, None
```
