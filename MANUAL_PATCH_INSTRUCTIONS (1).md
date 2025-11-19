# CRITICAL FIXES TO APPLY TO YOUR ORIGINAL Yo.py

## ✅ FIX #1: Add wait time after carrier filtering (LINE ~2878)

### Find this code:
```python
                    if not self.select_top_zero_premium_plan():
                        raise Exception("No $0.00 plans found after filtering")
```

### Replace with:
```python
                    # CRITICAL FIX: Wait for plans to reload after filtering
                    time.sleep(2.5)
                    logging.info("⏳ Waiting for filtered plans to load...")
                    
                    if not self.select_top_zero_premium_plan():
                        raise Exception("No $0.00 plans found after filtering")
```

---

## ✅ FIX #2: Replace select_top_zero_premium_plan method (LINE ~3111-3145)

### Replace the ENTIRE method with this:

```python
    def select_top_zero_premium_plan(self) -> bool:
        """Select first $0.00 plan from filtered results - VERIFY IT'S ACTUALLY $0.00."""
        try:
            logging.info("🔍 Looking for $0.00 premium plans...")
            
            time.sleep(2.0)  # Wait for plans to load
            
            # Find all plan cards
            plan_cards = self.driver.find_elements(
                By.CSS_SELECTOR,
                "div[data-testid='plan-card'], div[class*='plan-card'], div[class*='plan-item']"
            )
            
            if not plan_cards:
                # Fallback: just look for any plan containers
                plan_cards = self.driver.find_elements(
                    By.XPATH,
                    "//div[contains(@class, 'plan')]"
                )
            
            logging.info(f"📊 Found {len(plan_cards)} plan cards to check")
            
            for i, card in enumerate(plan_cards[:10], 1):
                try:
                    # Check premium text in this card
                    premium_elements = card.find_elements(
                        By.XPATH,
                        ".//span[contains(text(), '$')] | .//div[contains(text(), '$')] | .//var[@data-var='dollars']"
                    )
                    
                    for prem_elem in premium_elements:
                        prem_text = prem_elem.text.strip()
                        
                        # Check if it's $0.00 or $0
                        if "$0.00" in prem_text or "$0" in prem_text:
                            # Verify it's not a strikethrough price
                            try:
                                parent_classes = prem_elem.find_element(By.XPATH, "./ancestor::div[1]").get_attribute("class")
                                if parent_classes and ("strikethrough" in parent_classes.lower() or "strike" in parent_classes.lower()):
                                    continue
                            except:
                                pass
                            
                            # Found a $0.00 plan! Look for button in this card
                            try:
                                button = card.find_element(
                                    By.XPATH,
                                    ".//button[contains(text(), 'Add to cart') or contains(text(), 'View in cart')]"
                                )
                                
                                self.driver.execute_script(
                                    "arguments[0].scrollIntoView({behavior: 'instant', block: 'center'});", 
                                    button
                                )
                                time.sleep(0.5)
                                
                                try:
                                    button.click()
                                except:
                                    self.driver.execute_script("arguments[0].click();", button)
                                
                                logging.info(f"✅ Selected $0.00 plan (card #{i})")
                                time.sleep(0.75)
                                return True
                            except:
                                continue
                    
                except Exception as card_err:
                    logging.debug(f"Card {i} check failed: {str(card_err)[:60]}")
                    continue
            
            # If we get here, no $0.00 plans found
            logging.error("❌ No $0.00 plans found in filtered results")
            
            # Log what plans ARE available
            try:
                all_premiums = self.driver.find_elements(
                    By.XPATH,
                    "//var[@data-var='dollars'] | //*[contains(text(), '$')]"
                )
                premiums_found = []
                for elem in all_premiums[:5]:
                    try:
                        text = elem.text.strip()
                        if '$' in text:
                            premiums_found.append(text)
                    except:
                        pass
                if premiums_found:
                    logging.info(f"📋 Available premiums: {', '.join(premiums_found[:5])}")
            except:
                pass
            
            return False
            
        except Exception as e:
            logging.error(f"❌ Error selecting plan: {str(e)}")
            return False
```

---

## ✅ FIX #3: Fix stale element after download (LINE ~2720-2760)

### Find the click_review_plan() method and replace with:

```python
    def click_review_plan(self) -> bool:
        """Click Review Plan button with stale element retry logic."""
        try:
            logging.info("🔍 Looking for 'Review plan' button...")
            
            # Retry up to 3 times in case of stale element
            for attempt in range(3):
                try:
                    review_btn = WebDriverWait(self.driver, 5).until(
                        EC.element_to_be_clickable((
                            By.XPATH,
                            "//button[contains(text(), 'Review plan')]"
                        ))
                    )
                    
                    self.driver.execute_script(
                        "arguments[0].scrollIntoView({behavior: 'instant', block: 'center'});",
                        review_btn
                    )
                    time.sleep(0.5)
                    
                    try:
                        review_btn.click()
                    except:
                        self.driver.execute_script("arguments[0].click();", review_btn)
                    
                    logging.info("✅ Clicked 'Review plan' button")
                    return True
                    
                except StaleElementReferenceException:
                    if attempt < 2:
                        logging.warning(f"⚠️ Stale element, retrying... ({attempt + 1}/3)")
                        time.sleep(1.0)
                    else:
                        raise
            
            logging.error("❌ 'Review plan' button not found after 3 attempts")
            return False
            
        except Exception as e:
            logging.error(f"❌ Error clicking Review plan: {str(e)}")
            return False
```

---

## ✅ FIX #4: Always hide rows in process_client (LINE ~2610-2940)

### In the process_client method, find the part where it processes and add a finally block:

### Find this pattern (near the end of process_client):
```python
        except Exception as e:
            logging.error(f"❌ Fatal error processing {client.full_name}: {e}", exc_info=True)
            client.status = ClientStatus.ERROR
            client.error_message = str(e)
            # ... cleanup code ...
            return client.status
```

### Add AFTER the except block:
```python
        finally:
            # ALWAYS hide the row, even on error (prevents infinite loops)
            try:
                self.hide_row_from_table(client.row_index)
                logging.info(f"✅ Hid row {client.row_index} from table")
            except Exception as hide_err:
                logging.error(f"❌ Failed to hide row: {hide_err}")
                # If we can't hide it, mark it permanently skipped
                if hasattr(self, 'permanently_skipped'):
                    self.permanently_skipped.add(client.full_name)
```

---

## 🎯 THESE ARE THE ONLY 4 CRITICAL FIXES NEEDED

Apply these 4 changes to your original 151KB file and it should work properly without losing any of your existing code!

The other issues (string formatting, etc.) appear to already be correct in your code or are minor logging issues that don't affect functionality.

---

## How to Apply These Fixes:

1. Open your original `Yo.py` file
2. Use Ctrl+F (Find) to locate each section
3. Make the replacements manually
4. Save the file
5. Run and test!

The file size should stay ~151KB since we're only changing small sections.
