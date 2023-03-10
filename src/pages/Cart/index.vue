<template>
  <div class="cart">
    <template v-if="cart.length === 0">
      <cart-empty />
    </template>
    <div v-else class="container">
      <div class="cart__wrapper">
        <div class="cart__left">
          <div class="cart__card">
            <div class="card__label">
              <div class="card__label-text">Giỏ hàng</div>
              <div class="card__label-description">({{ cart.length }} sản phẩm)</div>
            </div>
            <div class="card__table">
              <table>
                <thead>
                  <tr>
                    <th>Sản phẩm</th>
                    <th>Giá</th>
                    <th>Số lượng</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="product in cart" :key="product.id">
                    <th>
                      <div class="product">
                        <div class="product__left">
                          <router-link
                            :to="{
                              name: 'ProductDetail',
                              params: {
                                slug: convertSlug(product.name),
                                categoryId: product.category_id,
                                productId: product.id,
                              },
                            }"
                            class="product__link"
                          >
                            <img :src="product.img_path" :alt="product.name" />
                          </router-link>
                        </div>
                        <div class="product__content">
                          <router-link
                            :to="{
                              name: 'ProductDetail',
                              params: {
                                slug: convertSlug(product.name),
                                categoryId: product.category_id,
                                productId: product.id,
                              },
                            }"
                            class="product__name"
                          >
                            <span>{{ product.name }}</span>
                          </router-link>
                          <button
                            @click.prevent="showDeleteModal(product)"
                            class="product__delete"
                          >
                            <i class="fas fa-trash"></i>
                          </button>
                        </div>
                      </div>
                    </th>
                    <td>
                      <div class="price__wrapper">
                        <span class="product__price--new">{{
                          Number(product.original_price)
                            | calDiscountPrice(product.discount)
                            | formatPrice
                        }}</span>
                        <div>
                          <span class="product__discount" v-show="product.discount > 0"
                            >-{{ product.discount }}%</span
                          >
                          <span
                            class="product__price--old"
                            v-show="product.discount > 0"
                            >{{ Number(product.original_price) | formatPrice }}</span
                          >
                        </div>
                      </div>
                    </td>
                    <td>
                      <div class="quantity__wrapper">
                        <div class="product__quantity">
                          <button
                            class="product__quantity-btn--minus"
                            @click="decreaseQuantity(product)"
                          >
                            -
                          </button>
                          <input
                            type="number"
                            min="1"
                            max="10000"
                            v-model="product.cart_quantity"
                            @input="setQuantity(product)"
                          />
                          <button
                            class="product__quantity-btn--plus"
                            @click="increaseQuantity(product)"
                          >
                            +
                          </button>
                        </div>
                      </div>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
        <div class="cart__right">
          <div class="cart__card">
            <div class="card">
              <div class="card__title">Mã giảm giá</div>
              <hr class="section__divider" />
              <div class="card__discount">
                <div class="card__input">
                  <input type="text" placeholder="Nhập mã giảm giá" />
                </div>
                <div class="card__btn">
                  <Button content="Áp dụng" radiusNone />
                </div>
              </div>
            </div>
          </div>
          <div class="cart__card">
            <div class="card">
              <table>
                <tbody>
                  <tr>
                    <th>Tạm tính</th>
                    <td>{{ Number(totalPrice) | formatPrice }}</td>
                  </tr>
                  <tr>
                    <th>Phí vận chuyển</th>
                    <td>{{ Number(transferFee) | formatPrice }}</td>
                  </tr>
                  <tr>
                    <th>Giảm giá</th>
                    <td>{{ Number(discount) | formatPrice }}</td>
                  </tr>
                  <tr class="total">
                    <th><strong>Thanh toán</strong></th>
                    <td>{{ Number(totalPaid) | formatPrice }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
          <Button
            @click="handleCheckout()"
            content="Tiến hành đặt hàng"
            radiusNone
            wFull
            size="large"
          />
        </div>
      </div>
      <modal-cart
        message="Bạn có chắc chắn muốn xóa sản phẩm này khỏi giỏ hàng không?"
        :show="false"
        ><div class="cart__actions">
          <Button content="Chắc chắn" @click="deleteProduct()" /><Button
            content="Hủy bỏ"
            theme="normal"
            @click="closeModal()"
          /></div
      ></modal-cart>
    </div>
  </div>
</template>

<script>
import Button from '@/components/Button/ButtonPrimary';
import { removeStorage } from '@/utils/storageWeb';
import { mapActions, mapGetters } from 'vuex';
import { orderApis } from '@/apis';
import CartEmpty from './CartEmpty';
import mixins from '@/mixins';
import ModalCart from '@/components/Modal/ModalCart.vue';

export default {
  name: 'Cart',
  mixins: [mixins],
  data: () => ({
    productDeleted: {},
  }),
  methods: {
    ...mapActions({
      setQuantity: 'auth/setProductQuantity',
      increaseQuantity: 'auth/increaseProductQuantity',
      decreaseQuantity: 'auth/decreaseProductQuantity',
      deleteProductFromCart: 'auth/deleteProductFromCart',
    }),

    // handle checkout
    async handleCheckout() {
      if (this.$store.getters['auth/isAuthenticated']) {
        try {
          let newCart = this.cart.map((item) => ({
            discount: item.discount,
            id: item.id,
            price: Math.ceil(
              item.original_price - (item.original_price * item.discount) / 100
            ),
            thumbnail: item.img_path,
            original_price: item.original_price,
            title: item.name,
            quantity: item.cart_quantity,
          }));

          let orderData = {
            carts: newCart,
            tempPrice: this.totalPrice,
            total: this.totalPaid,
          };

          let response = await orderApis.createOrder(orderData);
          if (response.status === 200) {
            await this.$router.push({
              name: 'CheckOut',
              params: {
                state: 'success',
                orderData: response.data,
              },
            });
            await this.$store.dispatch('auth/setCart', []);
            removeStorage('wolmartCart');
          }
        } catch {
          console.log('Something went wrong!');
          await this.$router.push({
            name: 'CheckOut',
            params: {
              state: 'fail',
            },
          });
        }
      } else {
        await this.$router.push({ name: 'Login' });
      }
    },
    showDeleteModal(product) {
      this.productDeleted = product;
      this.$modal.show('cart');
    },
    deleteProduct() {
      this.deleteProductFromCart(this.productDeleted);
      this.$modal.hide('cart');
    },
    closeModal() {
      this.$modal.hide('cart');
    },
  },
  computed: {
    ...mapGetters({ cart: 'auth/cart' }),
    totalPrice() {
      return this.cart.reduce(
        (previousValue, currentValue) =>
          previousValue +
          Math.ceil(
            currentValue.original_price -
              (currentValue.original_price * currentValue.discount) / 100
          ) *
            currentValue.cart_quantity,
        0
      );
    },
    transferFee() {
      return this.cart.some((currentValue) => currentValue.is_free_shipping == 0)
        ? 15000
        : 0;
    },
    discount() {
      return 0;
    },
    totalPaid() {
      return this.totalPrice + this.transferFee - this.discount;
    },
  },
  components: {
    Button,
    CartEmpty,
    ModalCart,
  },
};
</script>

<style lang="scss" scoped>
@import './Cart';
</style>
